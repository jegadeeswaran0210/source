#import "AppController.h"
2	#import "MASPreferencesWindowController.h"
3	#import "PreferencesGeneralViewController.h"
4	#import "PreferencesAdvancedViewController.h"
5	#import "SCTimeIntervalFormatter.h"
6	#import <SystemConfiguration/SystemConfiguration.h>
7	
8	NSString* const kSelfControlErrorDomain = @"SelfControlErrorDomain";
9	
10	@implementation AppController {
11		NSWindowController* getStartedWindowController;
12	}
13	
14	@synthesize addingBlock;
15	
16	- (AppController*) init {
17		if(self = [super init]) {
18	
19			defaults_ = [NSUserDefaults standardUserDefaults];
20	
21			NSDictionary* appDefaults = @{@"BlockDuration": @15,
22										  @"BlockStartedDate": [NSDate distantFuture],
23										  @"HostBlacklist": @[],
24										  @"EvaluateCommonSubdomains": @YES,
25										  @"IncludeLinkedDomains": @YES,
26										  @"HighlightInvalidHosts": @YES,
27										  @"VerifyInternetConnection": @YES,
28										  @"TimerWindowFloats": @NO,
29										  @"BlockSoundShouldPlay": @NO,
30										  @"BlockSound": @5,
31										  @"ClearCaches": @YES,
32										  @"BlockAsWhitelist": @NO,
33										  @"BadgeApplicationIcon": @YES,
34										  @"AllowLocalNetworks": @YES,
35										  @"MaxBlockLength": @1440,
36										  @"BlockLengthInterval": @15,
37										  @"WhitelistAlertSuppress": @NO,
38										  @"GetStartedShown": @NO};
39	
40			[defaults_ registerDefaults:appDefaults];
41	
42			self.addingBlock = false;
43	
44			// refreshUILock_ is a lock that prevents a race condition by making the refreshUserInterface
45			// method alter the blockIsOn variable atomically (will no longer be necessary once we can
46			// use properties).
47			refreshUILock_ = [[NSLock alloc] init];
48		}
49	
50		return self;
51	}
52	
53	- (NSString*)selfControlHelperToolPath {
54		static NSString* path;
55	
56		// Cache the path so it doesn't have to be searched for again.
57		if(!path) {
58			NSBundle* thisBundle = [NSBundle mainBundle];
59			path = [thisBundle pathForAuxiliaryExecutable: @"org.eyebeam.SelfControl"];
60		}
61	
62		return path;
63	}
64	
65	- (char*)selfControlHelperToolPathUTF8String {
66		static char* path;
67	
68		// Cache the converted path so it doesn't have to be converted again
69		if(!path) {
70			path = malloc(512);
71			[[self selfControlHelperToolPath] getCString: path
72											   maxLength: 512
73												encoding: NSUTF8StringEncoding];
74		}
75	
76		return path;
77	}
78	
79	- (IBAction)updateTimeSliderDisplay:(id)sender {
80		NSInteger numMinutes = floor([blockDurationSlider_ integerValue]);  
81	
82		// Time-display code cleaned up thanks to the contributions of many users
83	
84		NSString* timeString = [self timeSliderDisplayStringFromNumberOfMinutes:numMinutes];
85	
86		[blockSliderTimeDisplayLabel_ setStringValue:timeString];
87		[submitButton_ setEnabled: (numMinutes > 0) && ([[defaults_ arrayForKey:@"HostBlacklist"] count] > 0)];
88	}
89	
90	- (NSString *)timeSliderDisplayStringFromNumberOfMinutes:(NSInteger)numberOfMinutes {
91	    static NSCalendar* gregorian = nil;
92	    if (gregorian == nil) {
93	        gregorian = [[NSCalendar alloc] initWithCalendarIdentifier:NSGregorianCalendar];
94	    }
95	
96	    NSRange secondsRangePerMinute = [gregorian
97	                                     rangeOfUnit:NSSecondCalendarUnit
98	                                     inUnit:NSMinuteCalendarUnit
99	                                     forDate:[NSDate date]];
100	    NSUInteger numberOfSecondsPerMinute = NSMaxRange(secondsRangePerMinute);
101	
102	    NSTimeInterval numberOfSecondsSelected = (NSTimeInterval)(numberOfSecondsPerMinute * numberOfMinutes);
103	
104	    NSString* displayString = [self timeSliderDisplayStringFromTimeInterval:numberOfSecondsSelected];
105	    return displayString;
106	}
107	
108	- (NSString *)timeSliderDisplayStringFromTimeInterval:(NSTimeInterval)numberOfSeconds {
109	    static SCTimeIntervalFormatter* formatter = nil;
110	    if (formatter == nil) {
111	        formatter = [[SCTimeIntervalFormatter alloc] init];
112	    }
113	
114	    NSString* formatted = [formatter stringForObjectValue:@(numberOfSeconds)];
115	    return formatted;
116	}
117	
118	- (IBAction)addBlock:(id)sender {
119		[defaults_ synchronize];
120		if(([[defaults_ objectForKey:@"BlockStartedDate"] timeIntervalSinceNow] < 0)) {
121			// This method shouldn't be getting called, a block is on (block started date
122			// is in the past, not distantFuture) so the Start button should be disabled.
123			NSError* err = [NSError errorWithDomain:kSelfControlErrorDomain
124											   code: -102
125										   userInfo: @{NSLocalizedDescriptionKey: @"We can't start a block, because one is currently ongoing."}];
126			[NSApp presentError: err];
127			return;
128		}
129		if([[defaults_ arrayForKey:@"HostBlacklist"] count] == 0) {
130			// Since the Start button should be disabled when the blacklist has no entries,
131			// this should definitely not be happening.  Exit.
132	
133			NSError* err = [NSError errorWithDomain:kSelfControlErrorDomain
134											   code: -102
135										   userInfo: @{NSLocalizedDescriptionKey: @"Error -102: Attempting to add block, but no blocklist is set."}];
136	
137			[NSApp presentError: err];
138	
139			return;
140		}
141	
142		if([defaults_ boolForKey: @"VerifyInternetConnection"] && ![self networkConnectionIsAvailable]) {
143			NSAlert* networkUnavailableAlert = [[NSAlert alloc] init];
144			[networkUnavailableAlert setMessageText: NSLocalizedString(@"No network connection detected", "No network connection detected message")];
145			[networkUnavailableAlert setInformativeText:NSLocalizedString(@"A block cannot be started without a working network connection.  You can override this setting in Preferences.", @"Message when network connection is unavailable")];
146			[networkUnavailableAlert addButtonWithTitle: NSLocalizedString(@"Cancel", "Cancel button")];
147			[networkUnavailableAlert addButtonWithTitle: NSLocalizedString(@"Network Diagnostics...", @"Network Diagnostics button")];
148			if([networkUnavailableAlert runModal] == NSAlertFirstButtonReturn)
149				return;
150	
151			// If the user selected Network Diagnostics launch an assisant to help them.
152			// apple.com is an arbitrary host chosen to pass to Network Diagnostics.
153			CFURLRef url = CFURLCreateWithString(NULL, CFSTR("http://apple.com"), NULL);
154			CFNetDiagnosticRef diagRef = CFNetDiagnosticCreateWithURL(NULL, url);
155			CFNetDiagnosticDiagnoseProblemInteractively(diagRef);
156			return;
157		}
158	
159		[timerWindowController_ resetStrikes];
160	
161		[NSThread detachNewThreadSelector: @selector(installBlock) toTarget: self withObject: nil];
162	}
163	
164	- (void)refreshUserInterface {
165		if(![refreshUILock_ tryLock]) {
166			// already refreshing the UI, no need to wait and do it again
167			return;
168		}
169	
170		BOOL blockWasOn = blockIsOn;
171		blockIsOn = [self selfControlLaunchDaemonIsLoaded];
172	
173		if(blockIsOn) { // block is on
174			if(!blockWasOn) { // if we just switched states to on...
175				[self closeTimerWindow];
176				[self showTimerWindow];
177				[initialWindow_ close];
178				[self closeDomainList];
179			}
180		} 
181	    
182	    @end
183	    
