Snooze
=======================

Configurable snooze times.


History
-----------------------

Snooze is a fantastic example of, at least for me, an average tweak's development from idea to iteration to completion. When I first decided to develop Snooze, I assumed it had to begin in the MobileTimer private framework, that's why it wasn't possible to utilities like Flex, and why it didn't show up when I searched for it a few months ago (in a less experienced time). So, I dived in, and found a few great methods in the Alarm and xxxManager classes. Unfortunately, [%log](http://iphonedevwiki.net/index.php/Logos#.25log)'ing all of them wasn't very pretty. Only one was consistantely called:
   `+[Alarm isSnoozeNotification:arg1]`

But still, that sounded good. Here's what it spit out, after I snoozed an alarm:

`+[<Alarm: 0x199679bf8> isSnoozeNotification:<UIConcreteLocalNotification: 0x178134dc0>
	{
		fire date = Thursday, March 20, 2014 at 10:16:02 PM Eastern Daylight Time,
		time zone = (null),
		repeat interval = 0,
		repeat count = 0,
		next fire date = Thursday, March 20, 2014 at 10:16:02 PM Eastern Daylight Time,
		user info = {
				alarmId = "C4D933A4-0987-4DE2-91E0-51344EFB39B8";
				hour = 22;
				lastModified = "2014-03-21 02:06:17 +0000";
				minute = 7;
				repeatDay = "-1";
				revision = 1;
				soundType = 1;
		}
	}
]`

That seemed simple. But it was called more than once, maybe even dozens of times. And comparing all of those times would be exhausting. A simple implementation (reducing the "fire date" by 9, the default snooze time, and increasing it by SZADD_AMOUNT, my test amount) failed on all counts. So, I headed to some decompilations I had lying around for the Clock app, and SpringBoard itself. In a twist, I found this handy method, in a class I'd never even heard of:

	`-[SBClockDataProvider _handleAlarmSnoozedNotification:]`

The implementation went something like:

`-(void)_handleAlarmSnoozedNotification:(NSConcreteNotification *)notification {
	...
	... userInfo = [notification userInfo;]
	... alarm = [userInfo objectForKey:@"AlarmNotification"];
	... bulletin = [self _bulletinRequestForSnoozedAlarm:alarm];
	... timer = [[PCPersistentTimer alloc] init];
	[timer setFireDate:bulletin.fireDate];
	...
}`

So, I assumed overriding that to swap out the notification argument would work exactly the way I wanted, and it seemed to be the start of the trail. Here's what I logged out:

`-[<SBClockDataProvider: 0x1708710c0> _handleAlarmSnoozedNotification:NSConcreteNotification 0x170a43450
{
	name = SBApplicationClockAlarmSnoozedNotification;
	userInfo = {
		AlarmNotification = <UIConcreteLocalNotification: 0x17012c620>{
			fire date = Thursday, March 20, 2014 at 11:17:12 PM Eastern Daylight Time,
			time zone = (null),
			repeat interval = 0,
			repeat count = 0,
			next fire date = Thursday, March 20, 2014 at 11:17:12 PM Eastern Daylight Time,
			user info = {
				alarmId = "9D663886-D567-4DA0-A130-D1B7315FBF19";
				hour = 23;
				lastModified = "2014-03-21 03:07:34 +0000";
				minute = 8;
				repeatDay = "-1";
				revision = 1;
				soundType = 1;
			}
		};
	}
}
]`

`   [Snooze] Added 10
		to current time of 2014-03-21 03:22:01 +0000,

		to alter NSConcreteNotification 0x178c5a7c0 {
				name = SBApplicationClockAlarmSnoozedNotification; userInfo = {
				AlarmNotification = "<UIConcreteLocalNotification: 0x178134a00>{
					fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
					time zone = (null),
					repeat interval = 0,
					repeat count = 0,
					next fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
					user info = {
						alarmId = "49EB1B0B-63EA-46E7-99E2-F1AB6769F476";
						hour = 23;
						lastModified = "2014-03-21 03:21:25 +0000";
						minute = 22;
						repeatDay = "-1";
						revision = 1;
						soundType = 1;
					}
				};
			}
		}'s

		<UIConcreteLocalNotification: 0x178134a00>{
			fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
			time zone = (null),
			repeat interval = 0,
			repeat count = 0,
			next fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
			user info = {
				alarmId = "49EB1B0B-63EA-46E7-99E2-F1AB6769F476";
				hour = 23;
				lastModified = "2014-03-21 03:21:25 +0000";
				minute = 22;
				repeatDay = "-1";
				revision = 1;
				soundType = 1;
			}
		}

		and send NSConcreteNotification 0x178e4f000 {
			name = SBApplicationClockAlarmSnoozedNotification;
			userInfo = {
				AlarmNotification = <UIConcreteLocalNotification: 0x178134a00>{
					fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
					time zone = (null),
					repeat interval = 0,
					repeat count = 0,
					next fire date = Thursday, March 20, 2014 at 11:22:11 PM Eastern Daylight Time,
					user info = {
						alarmId = "49EB1B0B-63EA-46E7-99E2-F1AB6769F476\";
						hour = 23;
						lastModified = "2014-03-21 03:21:25 +0000";
						minute = 22;
						repeatDay = "-1";
						revision = 1;
						soundType = 1;
					}
				};
			}
		}`

But, that didn't work. It updated the timer for the bulletins (super easy to see on the lock screen), but it had no real effect on the actual timer or alarm. Some minor investigations into the decompilation made me feel as if there was some global "SBLocalNotificationSnoozeIntervalOverride" variable, but I couldn't find more than a single reference to that anywhere. Even in the simple snooze methods. It must've been a private value for actual system alerts. Moving on, I traced this method next:

	`-[SBClockDataProvider _handlePossibleAlarmNotificationUpdate:]`


Which ended me up in the PCPersistentTimer class, which was part of the sister framework PersistentConnection. Popping some logs from time-dependant PCPersistentTimer -init hook was nice-looking:

	`SpringBoard[30169]: [Snooze] Overriding preset time interval 9.999883 to be personalized 1395372853.630588...`

`   %hook PCPersistentTimer


	- (id)initWithTimeInterval:(double)arg1 serviceIdentifier:(id)arg2 target:(id)arg3 selector:(SEL)arg4 userInfo:(id)arg5 {
		if( sz_overrideInterval != 0.0) {
			NSLog(@"[Snooze] Overriding preset time interval %f to be personalized %f...", arg1, sz_overrideInterval);
			double snoozeInterval = sz_overrideInterval;
			sz_overrideInterval = 0.0;
			return %orig(snoozeInterval, arg2, arg3, arg4, arg5);
		}

		else {
			return %orig();
		}
	}

   %end`

But that had no additional effect on the codebase. Everything operated as it had before, meaning I was simply following the crumb trail *down*, instead of *up*. Without any other choice, I went back to the beautiful SBClockDataProvider, this time for -_publishAlarmsWithScheduledNotifications:

`without anything (set for 11:59, log 8 seconds later)
-[<SBClockDataProvider: 0x17867a500> _publishAlarmsWithScheduledNotifications:(
		<UIConcreteLocalNotification: 0x170322300>{
			fire date = Friday, March 21, 2014 at 12:08:08 AM Eastern Daylight Time,
			time zone = (null),
			repeat interval = 0,
			repeat count = 0,
			next fire date = Friday, March 21, 2014 at 12:08:08 AM Eastern Daylight Time,
			user info = {
				alarmId = "54D017B7-7ED8-468C-A23B-F0495CC85EC9";
				hour = 23;
				lastModified = "2014-03-21 03:58:15 +0000";
				minute = 59;
				repeatDay = \"-1\";
				revision = 1;
				soundType = 1;
			}
		}"
	)
]
--- last message repeated 1 time ---`

Uh-oh. This was down as well. The "fire date" was identical to the time snoozed. Hm... but inspecting the decompilation led me to a middle-man-looking method:
	`-[<SBClockDataProvider _nextAlarmForFeed:32 withNotifications]`

Let's see what it prints:

`-[<SBClockDataProvider: 0x170a7efc0> _nextAlarmForFeed:32 withNotifications:(
		<UIConcreteLocalNotification: 0x1783288e0>{
			fire date = Friday, March 21, 2014 at 12:16:01 AM Eastern Daylight Time,
			time zone = (null),
			repeat interval = 0,
			repeat count = 0,
			next fire date = Friday, March 21, 2014 at 12:16:01 AM Eastern Daylight Time,
			user info = {
				alarmId = \"E6A01559-F7A0-4A16-B016-CE65AC2EB953\";\n
				hour = 0;\n
				lastModified = \"2014-03-21 04:06:55 +0000\";\n
				minute = 7;
				repeatDay = "-1";
				revision = 1;
				soundType = 1;
			}
		}
	)
]`

Not bad! Looks like a very similar functionality to our parent %hook, back up there. Rubbing my hands together, I decided to create a global, static array that could hold all the alarmIds, and then, in this method, check to see if, in any of the notifications, there's one of those ids. If so, it would reduce the time by 9 minutes, and add my fake value (10 seconds). But, since this method is called several times, comparing the dates was tough... sometimes they were the same. Next call, though, they'd be already upped by 9 minutes, but they still had the 10 second addition from the previous call! What was happening?! After playing around with a bit too many stringy ideas for this method, I realized I was **still down**.

That's when SBLocalNotificationSnoozeIntervalOverride caught my eye again. It popped up in the stack trace for this method, *upwards*. But how did I miss that? Because it was surrounded by conditionals in the method I had started in. The full, jargony method had this one, serene line, that I underestimated:
	`_R0 = objc_msgSend(v7, "integerForKey:", CFSTR("SBLocalNotificationSnoozeIntervalOverride"));`

In essence, that translated into:
	`Make the snooze time additional equal to [[NSUserDefaults standardUserDefaults] integerForKey:SBLocalNotificationSnoozeIntervalOverride]`

It wasn't an override at all! It was just a SnoozeInterval, in the end, and it wanted everything but to be found. Logging all of the -integerForKey's of NSUserDefaults (the universal XML-wrapped quick-and-dirty storage system for iOS, useful for tiny values) made me realize Apple uses it a lot. Like, a whole lot. For tons of different things. I had no idea. Just for fun, I %hook'd the method, with a very simply body:


`   %hook NSUserDefaults

	- (NSInteger)integerForKey:(NSString *)defaultName {
		if ([defaultName isEqualToString:@"SBLocalNotificationSnoozeIntervalOverride"]) {
			return SZADD_AMOUNT;
		}

		else {
			return %orig();
		}
	}

   %end`

And you know what? It worked out. Perfectly. That's the only block of code that mattered. Hundreds of lines of code eliminated. Thousands written. The answer reducible to three.

Just another day in tweak land.

- Julian Weiss

---------------------------------------
[Creative Commons Attribution-NonCommercial 3.0 United States License](http://creativecommons.org/licenses/by-nc/3.0/us/) as of 2014:

	Creative Commons Attribution-NonCommercial 3.0 United States License
	Please visit above link for full license.
	Human-readable summary of your abilities has been transcribed below.

	You are free to:
	Share — copy and redistribute the material in any medium or format
	Adapt — remix, transform, and build upon the material
	The licensor cannot revoke these freedoms as long as you follow the license terms.

	Under the following terms:
	Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.
	NonCommercial — You may not use the material for commercial purposes.
	No additional restrictions — You may not apply legal terms or technological measures that legally restrict others from doing anything the license permits.

	Notices:
	You do not have to comply with the license for elements of the material in the public domain or where your use is permitted by an applicable exception or limitation.
	No warranties are given. The license may not give you all of the permissions necessary for your intended use. For example, other rights such as publicity, privacy, or moral rights may limit how you use the material.
