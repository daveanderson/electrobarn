Date: 2012-04-08
Title: Lesson 99 Part 3: Intro to Grand Central Dispatch (GCD)
Slug: lesson-99-part-3-grand-central-dispatch
Tags: code, objc, obj-c, objective-c, style, conventions, cocoa, gcd
Category: code

See also: [Part 1](/lesson-99-part-1-objective-c-and-cocoa-conventions.html) and [Part 2](/lesson-99-part-2-intro-to-blocks.html)

## Grand Central Dispatch

Grand Central Dispatch (GCD) is essentially a pool of threads available to perform work units, governed by a first-in, first-out queue (dispatch queue).

####Protecting access (using a queue)

Use a single queue and `dispatch_sync()` instead of a lock or other mechanism to protect a critical section.


[FMDB](https://github.com/ccgus/fmdb) has its own serial queue and uses `dispatch_sync()` to make sure that multiple threads can safely use the same database, and the queue ensures that only one thread's operation happens at a time.

    :::objc
    // from FMDatabaseQueue.m
	- (void)inDatabase:(void (^)(FMDatabase *db))block {
	    FMDBRetain(self);
	    
	    dispatch_sync(_queue, ^() {
	        
	        FMDatabase *db = [self database];
	        block(db);
	        
	        if ([db hasOpenResultSets]) {
	            NSLog(@"Warning: there is at least one open result set around after performing [FMDatabaseQueue inDatabase:]");
	        }
	    });
	    
	    FMDBRelease(self);
	}

I have a project that, using a background thread, downloads a new JSON dataset, converts it into a database and then when the database is up to date, I want to start use it without relaunching the app. I don't want to prevent the user from making queries while the *behind the scenes* update is happening.  So how do I do this safely?  I leveraged the queue!

    :::objc
    // added by DWA to safely swap database used by queue at runtime
	- (NSError *)switchToDatabaseWithPath:(NSString *)aPath {
	    // use the queue to ensure that we're not accessing the db as we switch
	    __block NSError *error = 0x00;
	    
	    dispatch_sync(_queue, ^() { 
	        
	        [_db close]; // close the old database
	        FMDBRelease(_db); // release it if not ARC
	        _db = 0x00; // set it to nil (FMDB convention)
	        
	        // delete old database and copy the new one so it has the same name
	        if([[NSFileManager defaultManager] removeItemAtPath:_path error:&error] &&
	           [[NSFileManager defaultManager] copyItemAtPath:aPath
	                                                   toPath:_path
	                                                    error:&error]){
	               NSLog(@"%@ successfully copied to %@", aPath, _path);
	           } else {
	               NSLog(@"Error description-%@ \n", [error localizedDescription]);
	               NSLog(@"Error reason-%@", [error localizedFailureReason]);
	           }
	        
	        _db = [FMDatabase databaseWithPath:_path]; // open my new database
	        FMDBRetain(_db); // retain it if not ARC
	        
	        if (![_db open]) {
	            NSLog(@"Could not create database queue for path %@", aPath);
	            FMDBRelease(self);
	            NSLog(@"Warning: failed to switch databases [FMDatabaseQueue switchToDatabaseWithPath:]");
	            error = [NSError errorWithDomain:@"FMDatabaseQueue" code:-1L userInfo:nil];
	        }
	    }); // end critical section!  All further queries using this queue will use the new database.
	    return error;
	}


####Fire and Forget

Use `dispatch_async()` to fire off an operation to complete in the background.

    :::objc
	- (void)loadDatabaseWithJSONFile:(NSString *)JSONFilePath {
	
	    dispatch_queue_t q = dispatch_queue_create("com.robotsandpencils.locationApp.loadDatabaseQueue", NULL);
	
	    FMDatabaseQueue *localQueue = self.queue; // use FMDB queue in the block but avoid using self
	
	    dispatch_async(q, ^{
	        // All this will happen on a background thread
			
	        NSError *jsonError = nil;
            NSData *jsonData = [NSData dataWithContentsOfFile:JSONFilePath 
                                                      options:NSDataReadingMappedAlways 
                                                        error:&jsonError];
            id json = [NSJSONSerialization JSONObjectWithData:jsonData 
                                                      options:NSJSONReadingMutableContainers 
                                                        error:&jsonError];
	        	        
			// error and version checking have been removed
	
	        NSString *documentsDirectory = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
	        NSString *databaseFilePath = [documentsDirectory stringByAppendingPathComponent:@"temp.db"];
	        
	        // create a new database and setup the tables and columns
	        FMDatabase *db = [RPDatabaseHelper databaseWithPath:databaseFilePath]; 
		
		    // the new location information
	        NSArray *locations = [json valueForKey:kDataLocationsKey]; 

	        int pk = 0;
	        if ([locations count] > 0) { // have locations to add to the database
		
	            [db beginTransaction];
	            NSLog(@"Begin loading database");
	            for (NSArray *location in locations) {
	
	                [RPDatabaseHelper insertLocation:location forIndex:pk intoDatabase:db];
	                pk += 1;
	            }
	
	            [db commit];
	            NSLog(@"End loading database");
	            
	            // can now call [localQueue switchToDatabaseWithPath:databaseFilePath]; to start using the new database
	        }
	        dispatch_release(q); // dispatch_queues aren't memory managed by ARC
	    });
	}


6. Asynchronous operations

Use nested `dispatch_async()` to fire off an operation and provide a block that will get called back on the main queue (main thread) to update the interface, for example.

    :::objc
    - (IBAction)showRenderedImage:(id)sender {
    	dispatch_queue_t q = dispatch_queue_create("com.robotsandpencils.locationApp.renderImageQueue", NULL);
        
        NSString *imageKey = self.imageKey; // avoid using self in the block
        UIImageView *imageView = self.imageView;
        
		// do my rendering on a different thread
        dispatch_async(q, ^{
            NSImageRep *image = [RPImageRenderer renderImageForKey:imageKey]; // computationally intensive
            // have image, need to display it using the main thread
            dispatch_async(dispatch_get_main_queue()), ^{
                
                // do something with the image on the main thread
                [imageView setImage:image];
                
            });
        dispatch_release(q);
        });
    }
    
That's great, but I really want to reuse this asynchronous renderer for more than one image.

    :::objc
    - (void)imageForKey:(NSString *)imageKey completion:(void (^)(UIImage *image))completion {
    
    	dispatch_queue_t q = dispatch_queue_create("com.robotsandpencils.locationApp.renderImageQueue", NULL);
        
		// do my rendering on a different thread
        dispatch_async(q, ^{
            NSImageRep *image = [RPImageRenderer renderImageForKey:imageKey]; // computationally intensive
            // have image, need to display it using the main thread
            dispatch_async(dispatch_get_main_queue()), ^{
                completion(image);
            });
            dispatch_release(q);
        });
    }

    - (IBAction)showRenderedBeehiveImage:(id)sender {
        UIImageView *imageView = self.beehiveImageView;
        [self imageForKey:@"beehive" completion:^(NSImageRep *image) {
            [imageView setImage:image];        
        }];
    }
    
    - (IBAction)showRenderedHoneyImage:(id)sender {
        UIImageView *imageView = self.honeyImageView;
        [self imageForKey:@"honey" completion:^(NSImageRep *image) {
            [imageView setImage:image];        
        }];
    }



####Avoiding the Watchdog 
Avoid the watchdog in `- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`

iOS has a watchdog timer that will kill apps that take too long to complete some operations.  Its handy to put lots of calls in `didFinishLaunchingWithOptions:` for things you need to do when the app launches.  But if you take too long, not only will you annoy your users by wasting their time, you risk having the watchdog terminate your app!

The app that inspired the snippets above has a bunch of content, images, and info that needs to be updated.  If all the downloading, database creating and other oprations happened sequentially within `didFinishLaunchingWithOptions:`, I'd surely trip the watchdog and get the app killed.

Minimize what happens in the main thread in this method by initiating operations and use GCD so that they complete on a background thread and `didFinishLaunchingWithOptions:` can end quickly.  

This can include

 * configuring and setting up analytics
 * downloading updates and content
 * loading user data
 
Your app will launch faster and your users will appreciate it.


###References:

* [ Concurrency Programming Guide](https://developer.apple.com/library/ios/#documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091)
* WWDC 2011: Session 308 - Blocks and Grand Central Dispatch in Practice
