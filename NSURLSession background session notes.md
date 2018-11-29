苹果员工评论：

Latest reply: Aug 18, 2015 4:49 AM by eskimo

Apple Staff (8,640 points)

eskimoAug 18, 2015 4:49 AM

When writing an app that uses NSURLSession’s background session support, it’s easy to get confused by three non-obvious artefacts of the development process:
* When you run your app from Xcode, Xcode installs the app in a new container, meaning that the path to your app changes.  This can confuse NSURLSession’s background session support.

Note This problem was fixed in iOS 9; if you encounter a problem with NSURLSession not handling a container path change in iOS 9 or later, please file a bug.

* Xcode’s debugging prevents the system from suspending your app.  So, if you run your app from Xcode, or you attach to the process some time after launch, and then move your app into the background, your app will continue executing in situations where the system would otherwise have suspended it.

* Similarly, the iOS Simulator does not accurately simulate app suspend and resume; this has worked in the past but it does not work in the iOS 8 or iOS 9 simulators (r. 16532261).

When doing in-depth testing of NSURLSession background sessions I recommend that you test on a real device, running your app from the Home screen rather than running it from Xcode.  This avoids all of the issues described above, resulting in a run-time environment that’s much closer to what your users will see.

If you encounter problems that you need to debug, you have two options:
* use logging — It’s important that your app have good logging support anyway, because otherwise it’ll be impossible to debug problems that only crop up in the field.  Once you’ve taken the time to create this logging, you can use it to debug problems during development.
* attach — If you have a specific problem that you must investigate with the debugger, you can run your app from the Home screen and then attach to the running process (via Xcode’s Debug > Attach to Process command) or use Wait for executable to be launched in the Info tab of the scheme editor.

**IMPORTANT** As mentioned above, the debugger prevents your app from suspending.  If that’s a problem, you can always detach and then reattach later on.

Finally, while bringing up a feature it’s often useful to start from a clean slate on launch; this prevents tasks from a previous debugging session from confusing the current debugging session.  You have a couple of options here:
* If you’re developing on the simulator, iOS Simulator > Reset Content and Settings is the quickest way to wipe the slate completely clean.
* On both the simulator and a real device, you can delete your app.  This will not only remove the app and everything in the app’s container, but will also remove any background sessions created by the app.
* You can also take advantage of NSURLSession’s -invalidateAndCancel method, which will cancel any running tasks and then invalidate the session.  There’s a few ways to use this:
    * During active development it might make sense for your app to call this on launch, guaranteeing that you start with a clean slate.
    * Alternatively, you could keep track of your app’s install path and call it on launch if the install path has changed.
    * You could have a hidden UI that invalidates the session.  This is helpful when you encounter a problem and want to know whether it’s caused by some problem with the NSURLSession persistent state.
