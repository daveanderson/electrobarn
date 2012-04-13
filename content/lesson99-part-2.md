Date: 2012-04-08
Title: Lesson 99 Part 2: Intro to Blocks
Slug: lesson-99-part-2-intro-to-blocks
Tags: code, objc, obj-c, objective-c, style, conventions, cocoa, blocks
Category: code

See also: [Part 1](/lesson-99-part-1-objective-c-and-cocoa-conventions.html) and [Part 3](/lesson-99-part-3-grand-central-dispatch.html)

## Blocks
What is a block?

Blocks encapsulate a unit of work (a segment of code) that can be executed at any time.  In other languages these they might be called "closures" or "lambdas".

If you're familiar with function pointers, just swap the `*` for a `^`.

### Function Pointer

	:::c
    float (*Multiply)(float, float) = NULL; // defining a function pointer
    float result = (*Multiply)(3.14, 2.0); // call function pointer

### Block

	:::objc
    int (^Multiply)(int, int) = ^(int num1, int num2) {
        return num1 * num2;
    }; // defining a block
    
    int result = Multiply(7, 4); // using a block
    
![Block Definition](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Blocks/Art/blocks.jpg)


As an argument to a method, a block acts as a callback or a way to customize a method or function.  Better than function pointers, blocks are defined where the method is invoked.  You don't have to add a method or function and the implementation is defined right where you're using it.  

	:::objc
	- (void)viewDidLoad {
	   [super viewDidLoad];
	    [[NSNotificationCenter defaultCenter] addObserver:self
	                                             selector:@selector(keyboardWillShow:)
	                                                 name:UIKeyboardWillShowNotification 
	                                               object:nil];
	}
	
	// at some other location in the same file separated by multiple methods
	- (void)keyboardWillShow:(NSNotification *)notification {
	    // Notification-handling code goes here.
	}

vs

	:::objc
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    [[NSNotificationCenter defaultCenter] addObserverForName:UIKeyboardWillShowNotification
	         object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification *notif) {
	             // Notification-handling code goes here - right where you're adding the observer
	    }];
	}

Blocks allow you to write code at the point of invocation that is executed later in the context of the method implementation.

Blocks allow access to local variables.  Rather than using callbacks requiring a data structure that embodies all the contextual information you need to perform an operation, you simply access local variables directly.


####Block basics

Blocks can have parameters and a return type (just like functions)
But they can also have access to local variables (better than functions)
    
	:::objc
	int multiplier = 7; // local variable
			
	int (^myBlock)(int) = ^(int num) {
	    return num * multiplier;
	};
			
	// if the block is defined as a variable, then you can used it just like a function
	printf("%d", myBlock(3));


####Favourite block API - UIView animateWithBlock:

	:::objc
	[UIView animateWithDuration:0.5f
					 animations:^{
					     // what animatable properties do I want to change
						 self.loginView.alpha = 0.0f;
					 } completion:^(BOOL finished) {
					     // what should happen when this animation is done
						 [self showWelcomeMessage]
					 }
	 ];	


####Getting data out of a block

Using FMDB as a front end to an sqlite database.  The latest version of FMDB uses a queue to manage access to a database and lets you provide a block to specify the block you need to happen.
   
	:::objc
	- (int)numberOfDatabaseEntries {
	    __block int count = 0;
	    [self.queue inDatabase:^(FMDatabase *db) {
	        FMResultSet *rs = [db executeQuery:@"SELECT COUNT(*) AS 'Count' from location_attributes"];
	        while ([rs next]) {
	            count = [rs intForColumn:@"Count"];
	        }
	    }];
		    
	    return count;
	}

I want to query the database to get find out how many items are in my database.  FMDB lets me provide a block for what I want to happen with the database (`db`) whose access is managed by the queue (`self.queue`).  My `count` variable is in scope for the block to read, however I want to modify the value of `count` and return it outside the block.  In order for changes to `count` to be available, I have to mark it with the `__block` storage type modifier.

4. Avoid retain issues
 
I love to use property within a class so that I know that I'm using accessors and not touching the ivar directly, so my code is littered with `self.thisProperty` and `self.thatProperty`.  However, in order for `self.thisProperty` accessor to be triggered from within the block, `self` would have to be retained by the block, causing an unbalanced retain which will turn into a memory leak!  Solution? Use a local variable.

    :::objc
	- (NSArray *)locationsForCityOrderByName:(NSString *)city {
		
	    __block NSMutableArray *locations = [[NSMutableArray alloc] init];
		    
	    NSString *searchString = self.searchString;
		
	    [self.queue inDatabase:^(FMDatabase *db) {
	        FMResultSet *rs = nil;
            rs = [db executeQuery:@"SELECT location_attributes.* FROM location_attributes, location_fts \
                  WHERE location_attributes.city=? \
                  AND location_fts.id=location_attributes.id \
                  AND location_fts.name_type_address LIKE ? \
                  ORDER BY location_attributes.locationName", 
                  city, 
                  [NSString stringWithFormat:@"%%%@%%%", searchString]];
	                  
	        while ([rs next]) {
		            
	            RPLocation *location = [RPLocation locationFromDictionary:[rs resultDict]];
	            if (!location) {
	                continue;
	            }
	            [locations addObject:location];
	        }
	    }];
	    return locations;    
	}

####Summary

   A block is an anonymous inline collection of code that:

	  * Has a typed argument list just like a function
	  * Has an inferred or declared return type
	  * Can capture state from the lexical scope within which it is defined
	  * Can optionally modify the state of the lexical scope (via `__block`)
	  * Can share the potential for modification with other blocks defined within the same lexical scope
	  * Can continue to share and modify state defined within the lexical scope (the stack frame) after the lexical scope (the stack frame) has been destroyed

   Because blocks are portable and anonymous objects encapsulating a unit of work that can (often) be performed asynchronously, they are a central feature of Grand Central Dispatch

###References:

 * [ A Short Practical Guide to Blocks](https://developer.apple.com/library/ios/#featuredarticles/Short_Practical_Guide_Blocks/_index.html)
 * [ Getting Started with Blocks](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/Blocks/Articles/bxGettingStarted.html#//apple_ref/doc/uid/TP40007502-CH7-SW1)