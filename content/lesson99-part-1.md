Date: 2012-04-07
Title: Lesson 99 Part 1: Objective-C and Cocoa Conventions
Slug: lesson-99-part-1-objective-c-and-cocoa-conventions
Tags: code, objc, obj-c, objective-c, style, conventions, cocoa
Category: code

Last month I was asked to present at [iPhone Dev Camp #6](http://calgary.iphonedevcamps.org/iphone-dev-camp-6).  I gave the organizer a few topics that I'd had some recent experience with that I thought would be relevant and three were selected.  When I checked the Dev Camp website, however, I could see that I was listed as presenting "Lesson 99".  So I decided to run with Lesson 99 as a theme.  This is my compilation of notes and opinions as influenced by search results indicating that Lesson 99 was [Beekeeper Or Bee-haver ](http://basicbeekeeping.blogspot.ca/2011/03/lesson-99-beekeeper-or-bee-haver.html).  This is part 1.  See [Part 2](/lesson-99-part-2-intro-to-blocks.html) and [Part 3](/lesson-99-part-3-grand-central-dispatch.html) for the other topics.

### Lesson 99 – Beekeeper or Bee-haver
A BEE-HAVER is someone who can say they “HAVE” bees but they do not want beekeeping to consume their time or interest, so they spend little to no time keeping bees, they simply have bees. That’s certainly one approach.

Then there are those who want to evolve from just having bees to truly doing all they can to make sure their bees are as healthy as possible.

![Beekeeper](http://lh6.ggpht.com/_qz9aE02ggbw/TY1ziX77DhI/AAAAAAAACaU/5IbRQKLI9gc/s1600/Overwintered+hive+2011%5B2%5D.jpg)


### Objective-C and Cocoa Conventions
*AKA "Stop making your keyboard cry!"*

![Cocoa](https://devimages.apple.com.edgekey.net/technologies/mac/images/cocoa_cup.jpg)

I'm going to assume some familiarity with Obj-C & Cocoa and look at some basic tips for improving your code  (Obj-C is the language, Cocoa is the application frameworks.)

Why do I care about this?  Because when you inherit one of my projects I want 
  
* the code to be self documenting (not a bunch obsolete comments)
* reusable – copy bits into other projects
* flexible - readily adaptable to meet changing requirements

You should be able to skim my code and quickly see what happens in each class.

#### Method Names
* spacing - recommend following 's convention.  That way you match Xcode snippets, sample code, etc.  (Apple isn't perfect in following their own convention, but there's less friction if you just follow their lead.)
* method signature is the name of a message sent to an object (sent via `obj_msg_send()`)
* method name "selects" a method implementation, so often referred to as **selector**
* methods with multiple parameters should have the method name interleaved with the parameters.  This lets the method name describe the parameters.  Selectro name includes all parts of the name, including the colons.
   
 **Method Signature:**
   
	:::objc
	- (void)insertHive:(id)beehive atIndex:(NSUInteger)index

 **Selector**

	:::objc
	insertHive:atIndex:
   
 **Usage**
   
	:::objc
	[aFarmersField insertHive:whiteBeehive atIndex:5];
         
 **Syntactically permissble but you're making your keyboard cry**

	:::objc
	- (void)insert:(id)something :(NSUInteger)index
   
	insert::
         
	[anObject insert:newObject :5];
         
 This fundamental difference in method naming in ObjC may be difficult to grok at the beginning but improves code readability and should give you hints as to what parameters are required.  The behaviour should definitely be clear.

----

#####Accessor Methods
Getters should be the name of the ivar/property, and shouldn't include the term "get".
   
 **Bravo!** 

	:::objc
	NSArray *hives = [NSArray array];
         
	[aFarmersField setHives:hives];
	aFarmersField.hives = hives; // property assignment calls setter setHives:
     
	hives = [aFarmersField hives];
	hives = aFarmersField.hives; // use of property calls getter
         
 **Keyboard Rain**
         
	:::objc
	[aFarmersField getHives];

----

#####We Read More Than We Write
Be descriptive.  Objective-C and Cocoa are designed to read well.
   
 **Obj-C Pro**
   
	destinationSelection
	setBackgroundColor:
	rowIndex
	name
	RPUpdateLocationInformationOperationDidCompleteNotification
         
 **Tears make the keys slippery**
   
	destSel
	setBkgdColor:
	i
	szName
	downloadNotification
   
    methodNamesUseCamelCase:additionalParameter:
    RPSmokeTheBees() && RPBeeHiveWidth

----
     
##### New and Copy
Beware of methods starting with `new`, `copy`, `create` – they imply specific memory management and may confuse ARC (and other developers!) Note: `create` has implications for CoreFoundation and CoreGraphics code (not necessarily Obj-C).
    
![Method Declaration Syntax](https://developer.apple.com/library/ios/referencelibrary/GettingStarted/Learning_Objective-C_A_Primer/Art/method_decl.jpg)

#### Actions
The things that happen when you tap a button. The method name is the response, not the state.

When you connect a button between a xib and a method its essentially the same as calling these two methods
 
	:::objc
	[myButton setAction:@selector(processHoney:)];
	[myButton setTarget:anObject];
            
When the button is tapped, this method is getting called
  
	[anObject performSelector:@selector(processHoney:) withObject:myButton];
            
This message passing is decided at run time and is totally dynamic, so there's no need to create static associations between a button and a method.  An `IBAction`isn't an button or event handler!  (IBAction is actually `#define IBAction void`)

----

##### IBActions
The identifier `IBAction` indicates the action that *will happen* not the action that *did happen*.
      
**Über-clear**

	:::objc
	- (IBAction)showSettingsPopover:(id)sender;
	- (IBAction)checkUserLoginCredentials:(id)sender;
	- (IBAction)smokeTheHive;
	- (IBAction)hideLicenseAgreement;

**The home row weeps**

	:::objc
    - (IBAction)blueButtonPressed:(id)sender;
    - (IBAction)loginTapped:(id)sender;
    - (IBAction)tabButtonTouchUpInside:(id)sender;
    - (IBAction)agree;
            
Why is this a big deal?
      
Designs change.

Buttons change.

Actions change.
    
By defining the state in which you *expect* a method to be used, you're immediately and artificially limiting any other use.  You're also giving *no* cues as to the behaviour of the code in the method.  
      
What happens when the blue button is pressed?  Do you care that the button, 3 revisions ago, used to be blue?
      
Will the `sender` always be blue? Login? a TouchUpInside?
      
If you name it per proper convention, then its no big deal when 3 different buttons are all connected to `smokeTheHive` or when the button that triggers `showSettingsPopover:` gets changed 36 times over the course of the project - their actions *that will happen* are always clear.

#### Class Prefixes

    RPLoginViewController
    RPHive
    RPDroneBee
   
    LoginViewController
    Hive
    DroneBee
                  
 * Recommended to avoid collisions
 * Redundant characters and seem less readable
 * Great for refactoring and find/replace because the origin is totally clear
   
#### Constants

Use enumerations for groups of related constants that have integer values
 
	:::objc
    typedef enum {
        RPMapListSegmentedControlMapIndex = 0,
        RPMapListSegmentedControlListIndex,
    } RPMapListSegmentedControl;

Use `const` to create constants for floating point values

	:::objc
    // Common.h
    extern const float RPImageOverlayViewRadiusMeters;
	        
    // RPImageOverlayView.m
	        
    const float RPImageOverlayViewRadiusMeters = 100.0f;

In general don't use the `#define` preprocessor command to create constants.  (e.g They won't be available in the debugger.) 

Define constants for strings used for notification names and dictionary keys.  By using string constants the compiler can perform verification.
 
    :::objc
    // Common.h
    extern NSString * const RPLocationsDatabaseName;
    extern NSString * const RPHelpVersionKey;

	// RPSecretLocations.m
	NSString * const RPLocationsDatabaseName = @"secretLocations.sqlite";
    NSString * const RPHelpVersionKey = @"42";

#### Extensions vs Categories

#####Categories
Add methods to an existing class or break up an implementation into multiple files. (Can't add ivars or properties)
 
	:::objc
    // NSString+RPAdditions.h
             
    @interface NSString (RPAdditions)
             
    + (NSString *)RP_stringFromBase64Data:(NSData *)data;
             
    // this allows [NSString RP_stringFromBase64Data:base64Data];

    @end
       
#####Extensions
Private implementation details
 
	:::objc
     @interface RPHive ()
             
     // private methods
     - (NSNumber *)hiveWeight;
     - (NSNumber *)smokePenetrationFactor;
             
     // private properties
     @property (strong, nonatomic) NSNumber *numberOfDrones;
     @property (nonatomic) NSNumber *numberOfWorkers;
     @property (nonatomic) NSString *hiveIdentifier;
             
     @end
             
     @implementation RPHive {
         // private ivars
         BOOL _queenPresent;
         NSNumber *width;
         NSNumber *height;
         NSNumber *depth;
     }
             
     @synthesize numberOfDrones = _numberOfDrones;
     @synthesize numberOfWorkers = _numberOfWorkers;
     @synthesize hiveIdentifier = _hiveIdentifier;

     // implementation
             
     @end

#### Literals 
For NSDictionary, NSArray and NSNumber (available in Xcode 4.4)

 * `@"moof";` is a literal for `[NSString stringWithCString:"moof" encoding:NSUTF8StringEncoding];`

Soon we'll have
     
 * `@42;` instead of `[NSNumber numberWithInt:42];`
 * `@3.141592653;` instead of `[NSNumber numberWithDouble:3.141592653];`
 * `@'L';` instead of `[NSNumber numberWithChar:'L'];`
 * `@YES;` instead of `[NSNumber numberWithBool:YES];`
 * `@2.718f;` instead of `[NSNumber numberWithFloat:2.719f];`
 * `@256u;` instead of `[NSNumber numberWithUnsignedInt:256u];`     
     
Then in combination with the new NSArray literal
     
 * `NSArray *easyArray = @[@"foo", @'Q', @100.0f, @YES];`
     
Similary for NSDictionary
     
 * `NSDictionary *simpleDictionary = @{ @"redKey": @"red", @42: @NO, @"blueKey": @22.0f };`
     
And a non-literal improvement coming in Xcode4.4 is that `@synthesize` becomes optional!

#### ARC 
Use Automatic Reference Counting instead of MRC/MRR - Manual Reference Counting / Manual Retain Release

* Use it!  Converting most projects is a 30 minute process (or less)
* It's less code (less is always better - less room for mistakes)
* It's more efficient (leaking is harder, memory footprint is smaller)
* Its less work – my knowledge of effective retain/release patterns are like old war medals – well earned and give great perspective, but not something I want to use all the time.
    

###References: 

 * [ Coding Guidelines for Cocoa](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html)
 * [ Object-Oriented Programming with Objective-C](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/OOP_ObjC/Introduction/Introduction.html)
 * [Cocoa Style for Objective-C: Part I](http://cocoadevcentral.com/articles/000082.php)
 * [Cocoa Style for Objective-C: Part II](http://cocoadevcentral.com/articles/000083.php)
 * [ The Objective-C Programming Language](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html)
 * [CocoaHeads • Objective-C literals for NSDictionary, NSArray, and NSNumber](http://cocoaheads.tumblr.com/post/17757846453/objective-c-literals-for-nsdictionary-nsarray-and)
 * [ Categories and Extensions](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ObjectiveC/Chapters/ocCategories.html#//apple_ref/doc/uid/TP30001163-CH20-SW1)

