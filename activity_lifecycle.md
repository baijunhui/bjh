# bjh

Activity 生命周期学习

	这个是经典的android的生命周期的流程
	onCreate() -> onStart() -> onResume() -> onPause() -> onStop() -> onDestroy()
	or 
	onStop() -> onReStart() -> onStart() ->onResume()

	下面我们来分析每一项是怎样实现的，怎样回调回来实现的

    分析之前，我们先来介绍几个名词
    1)activity ： app的最小单元，用来显示界面的，四大组件之一，最重要的一个组件
    2)Instrumentation ： 用来监控程序和系统之间的交互的
    3)ActivityManagerService ： 它是android中最重要的服务，主要负责四大组件的启动，切换，调度及应用进程的调度等工作，其职责与进程管理里的调度类似
    4)ActivityThread ： 主线程也就是UI线程,activity的生命周期的实现都是在这个里面的

	1、我们启动一个activity是从startActivity(Intent)开始的，所以我们从startActivity方法开始追踪

    	public void startActivity(Intent intent) {
            this.startActivity(intent, null);
        }

        public void startActivity(Intent intent, @Nullable Bundle options) {
            if (options != null) {
                startActivityForResult(intent, -1, options);
            } else {
                // Note we want to go through this call for compatibility with
                // applications that may have overridden the method.
                startActivityForResult(intent, -1);
            }
        }

        public void startActivityForResult(Intent intent, int requestCode) {
            startActivityForResult(intent, requestCode, null);
        }

        public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
            if (mParent == null) {
                Instrumentation.ActivityResult ar =
                    mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
                。。。
        }

        startActivity方法最终调用了activity的startActivityForResult方法，是不是很熟悉，我们有时候如果想要回调的之前界面，就是使用startActivityForResult方法的，比如，我们使用相机或者照片选择照片时，就是使用回调到原来页面的，并把数据带回来。下面我们继续分析主要代码

    2、Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);

        这个是调用了Instrumentation的execStartActivity方法，看名字我们就可以知道，这个是执行启动activity的
        继续看

    3、Instrumentation的execStartActivity方法

        public ActivityResult execStartActivity(
                Context who, IBinder contextThread, IBinder token, Activity target,
                Intent intent, int requestCode, Bundle options) {
            。。。
            try {
                intent.migrateExtraStreamToClipData();
                intent.prepareToLeaveProcess();
                int result = ActivityManagerNative.getDefault()
                    .startActivity(whoThread, who.getBasePackageName(), intent,
                            intent.resolveTypeIfNeeded(who.getContentResolver()),
                            token, target != null ? target.mEmbeddedID : null,
                            requestCode, 0, null, options);
                //检查activity
                checkStartActivityResult(result, intent);
            } catch (RemoteException e) {
            }
            return null;
        }

        这里的关键代码是try里面的代码，可以看出try里的代码，ActivityManagerNative.getDefault()拿到ActivityManagerService的远程IBinder（代理），也就是AMS,这个里面涉及了进程间的通信，具体的可以查阅相关资料，下面我们来看AMS里的startActivity方法

    4、AMS里面的startActivity方法
        @Override
        public final int startActivity(IApplicationThread caller, String callingPackage,
                Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                int startFlags, ProfilerInfo profilerInfo, Bundle options) {
            return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, options,
                UserHandle.getCallingUserId());
        }

        @Override
        public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
                Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
            enforceNotIsolatedCaller("startActivity");
            userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                    false, ALLOW_FULL_ONLY, "startActivity", null);
            // TODO: Switch to user app stacks here.
            return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                    resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                    profilerInfo, null, null, options, userId, null, null);
        }

        我们可以看到startActivity方法里，只是调用了其他方法，没有做任何处理，我们看下startActivityAsUser方法，里面调用了ActivityStackSupervisor的mayWait方法，这个里面保存了activity所需要的stack栈的信息 

        ActivityStackSupervisor从名字我们可以看出，这是个activity栈的监督者，监督activity栈的工作情况的。

    5、ActivityStackSupervisor。startActivityMayWait方法

        final int startActivityMayWait(IApplicationThread caller, int callingUid,
                String callingPackage, Intent intent, String resolvedType,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                IBinder resultTo, String resultWho, int requestCode, int startFlags,
                ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
                Bundle options, int userId, IActivityContainer iContainer, TaskRecord inTask) {

            	。。。

                int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                        voiceSession, voiceInteractor, resultTo, resultWho,
                        requestCode, callingPid, callingUid, callingPackage,
                        realCallingPid, realCallingUid, startFlags, options,
                        componentSpecified, null, container, inTask);

                。。。

                return res;
            }
        }

        final int startActivityLocked(IApplicationThread caller,
                Intent intent, String resolvedType, ActivityInfo aInfo,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                IBinder resultTo, String resultWho, int requestCode,
                int callingPid, int callingUid, String callingPackage,
                int realCallingPid, int realCallingUid, int startFlags, Bundle options,
                boolean componentSpecified, ActivityRecord[] outActivity, ActivityContainer container,
                TaskRecord inTask) {

            。。。


            ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                    intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                    requestCode, componentSpecified, this, container, options);
            if (outActivity != null) {
                outActivity[0] = r;
            }

            final ActivityStack stack = getFocusedStack();
            。。。


            doPendingActivityLaunchesLocked(false);

            err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, true, options, inTask);

            if (err < 0) {
                // If someone asked to have the keyguard dismissed on the next
                // activity start, but we are not actually doing an activity
                // switch...  just dismiss the keyguard now, because we
                // probably want to see whatever is behind it.
                notifyActivityDrawnForKeyguard();
            }
            return err;
        }



        //设置启动模式
        final int startActivityUncheckedLocked(ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
            final Intent intent = r.intent;
            final int callingUid = r.launchedFromUid;

            // In some flows in to this function, we retrieve the task record and hold on to it
            // without a lock before calling back in to here...  so the task at this point may
            // not actually be in recents.  Check for that, and if it isn't in recents just
            // consider it invalid.
            if (inTask != null && !inTask.inRecents) {
                Slog.w(TAG, "Starting activity in task not in recents: " + inTask);
                inTask = null;
            }

            final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
            final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
            final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;

            int launchFlags = intent.getFlags();
            ...

            final boolean launchTaskBehind = r.mLaunchTaskBehind
                    && !launchSingleTask && !launchSingleInstance
                    && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;


            ...

            
            ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
            targetStack.mLastPausedActivity = null;
            targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
            if (!launchTaskBehind) {
                // Don't set focus on an activity that's going to the back.
                mService.setFocusedActivityLocked(r);
            }
            return ActivityManager.START_SUCCESS;
        }
        上面一段代码主要是模式的设置，很复杂，可以自己研究下

    6、下面看下ActivityStack的startActivityLoacked方法

        final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
            。。。

            // Slot the activity into the history stack and proceed
            if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                    new RuntimeException("here").fillInStackTrace());
            //添加r到最顶层
            task.addActivityToTop(r);
            task.setFrontOfTask();

            。。。

            if (doResume) {
                mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
            }
        }

    7、我们分析前面的可以知道，doResume是true，所以还是调用了ActivityStackSupervisor的resumeTopActivitiesLocked方法

        boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
            if (targetStack == null) {
                targetStack = getFocusedStack();
            }
            // Do targetStack first.
            boolean result = false;
            //是false
            if (isFrontStack(targetStack)) {
                result = targetStack.resumeTopActivityLocked(target, targetOptions);
            }
            for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
                final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
                for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                    final ActivityStack stack = stacks.get(stackNdx);
                    if (stack == targetStack) {
                        // Already started above.
                        continue;
                    }
                    //这个才是true
                    if (isFrontStack(stack)) {
                        stack.resumeTopActivityLocked(null);
                    }
                }
            }
            return result;
        }

    8、我们可以看到，又调回了ActivityStack的resumeTopActivityLocked方法

        final boolean resumeTopActivityLocked(ActivityRecord prev) {
            return resumeTopActivityLocked(prev, null);
        }

        final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
            if (inResumeTopActivity) {
                // Don't even start recursing.
                return false;
            }

            boolean result = false;
            try {
                // Protect against recursion.
                inResumeTopActivity = true;
                result = resumeTopActivityInnerLocked(prev, options);
            } finally {
                inResumeTopActivity = false;
            }
            return result;
        }

        final boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
            
            。。。

            //我们从topRunningActivityLocked方法里可以知道，这个拿到的next是我们要启动的activity，因为我们之前已经添加到栈顶了
            // Find the first activity that is not finishing.
            ActivityRecord next = topRunningActivityLocked(null);

            。。。

            //next不为null
            if (next == null) {
                // There are no more activities!  Let's just start up the
                // Launcher...
                ActivityOptions.abort(options);
                if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: No more activities go home");
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                // Only resume home if on home display
                final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                        HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
                return isOnHomeDisplay() &&
                        mStackSupervisor.resumeHomeStackTask(returnTaskType, prev);
            }

            //mResumedActivity还是之前的activity，所以和next不相等，所以是false
            // If the top activity is the resumed one, nothing to do.
            if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                        mStackSupervisor.allResumedActivitiesComplete()) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                mWindowManager.executeAppTransition();
                mNoAnimActivities.clear();
                ActivityOptions.abort(options);
                if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Top activity resumed " + next);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();

                // Make sure to notify Keyguard as well if it is waiting for an activity to be drawn.
                mStackSupervisor.notifyActivityDrawnForKeyguard();
                return false;
            }

            。。。

            // The activity may be waiting for stop, but that is no longer
            // appropriate for it.
            mStackSupervisor.mStoppingActivities.remove(next);
            mStackSupervisor.mGoingToSleepActivities.remove(next);
            next.sleeping = false;
            mStackSupervisor.mWaitingVisibleActivities.remove(next);

            if (DEBUG_SWITCH) Slog.v(TAG, "Resuming " + next);

            // If we are currently pausing an activity, then don't do anything
            // until that is done.
            if (!mStackSupervisor.allPausedActivitiesComplete()) {
                if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG,
                        "resumeTopActivityLocked: Skip resume: some activity pausing.");
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return false;
            }

            。。。

            //开始暂停前一个activity的显示，因为mResumeActivity不为null，所以执行了startPausingLocked方法,最好返回true，pausing是true，所以， //不执行到下面的代码了,当当前activity已经暂停的时候，mResumedActivity为null了，所以不会执行该段代码，跳过了

            // We need to start pausing the current activity so the top one
            // can be resumed...
            boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
            boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
            if (mResumedActivity != null) {
                pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
                if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
            }
            if (pausing) {
                。。。
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }

            。。。

            //next是当前activity界面的Record
            ActivityStack lastStack = mStackSupervisor.getLastStack();
            if (next.app != null && next.app.thread != null) {

                。。。

                mResumedActivity = next;
                next.task.touchActiveTime();
                mService.addRecentTaskLocked(next.task);
                mService.updateLruProcessLocked(next.app, true, null);
                updateLRUListLocked(next);
                mService.updateOomAdjLocked();

                。。。

                try {
                    // Deliver all pending results.
                    ArrayList<ResultInfo> a = next.results;
                    if (a != null) {
                        final int N = a.size();
                        if (!next.finishing && N > 0) {
                            if (DEBUG_RESULTS) Slog.v(
                                    TAG, "Delivering results to " + next
                                    + ": " + a);
                            next.app.thread.scheduleSendResult(next.appToken, a);
                        }
                    }

                    if (next.newIntents != null) {
                        next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                    }

                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY,
                            next.userId, System.identityHashCode(next),
                            next.task.taskId, next.shortComponentName);

                    next.sleeping = false;
                    mService.showAskCompatModeDialogLocked(next);
                    next.app.pendingUiClean = true;
                    next.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_TOP);
                    next.clearOptionsLocked();
                    next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                            mService.isNextTransitionForward(), resumeAnimOptions);

                    mStackSupervisor.checkReadyForSleepLocked();

                    if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Resumed " + next);
                } catch (Exception e) {
                    。。。            }

                // From this point on, if something goes wrong there is no way
                // to recover the activity.
                try {
                    next.visible = true;
                    completeResumeLocked(next);
                } catch (Exception e) {
                    。。。
                }
                next.stopped = false;

            } else {
                // Whoops, need to restart this activity!
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else {
                    if (SHOW_APP_STARTING_PREVIEW) {
                        mWindowManager.setAppStartingWindow(
                                next.appToken, next.packageName, next.theme,
                                mService.compatibilityInfoForPackageLocked(
                                        next.info.applicationInfo),
                                next.nonLocalizedLabel,
                                next.labelRes, next.icon, next.logo, next.windowFlags,
                                null, true);
                    }
                    if (DEBUG_SWITCH) Slog.v(TAG, "Restarting: " + next);
                }
                if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Restarting " + next);
                mStackSupervisor.startSpecificActivityLocked(next, true, true);
            }

            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        上面代码比较多，我...了一部分代码，加了一些中文注释，从上面我们可以知道，之前的mOnResumeActivity不为null，所以执行了pasuse状态，下面是执行了startPausingLocked方法

        startPausingLocked方法

        final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
            ...
            //prev是上一个activity，所以里面包含了app,thead
            ActivityRecord prev = mResumedActivity;
            。。。
            
            mResumedActivity = null;
            mPausingActivity = prev;
            mLastPausedActivity = prev;
            mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                    || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
            prev.state = ActivityState.PAUSING;
            。。。

            if (prev.app != null && prev.app.thread != null) {
                if (DEBUG_PAUSE) Slog.v(TAG, "Enqueueing pending pause: " + prev);
                try {
                    EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                            prev.userId, System.identityHashCode(prev),
                            prev.shortComponentName);
                    mService.updateUsageStats(prev, false);
                    prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                            userLeaving, prev.configChangeFlags, dontWait);
                } catch (Exception e) {
                    // Ignore exception, if process died other code will cleanup.
                    Slog.w(TAG, "Exception thrown during pause", e);
                    mPausingActivity = null;
                    mLastPausedActivity = null;
                    mLastNoHistoryActivity = null;
                }
            } else {
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }

            // If we are not going to sleep, we want to ensure the device is
            // awake until the next activity is started.
            if (!mService.isSleepingOrShuttingDown()) {
                mStackSupervisor.acquireLaunchWakelock();
            }

            if (mPausingActivity != null) {
                // Have the window manager pause its key dispatching until the new
                // activity has started.  If we're pausing the activity just because
                // the screen is being turned off and the UI is sleeping, don't interrupt
                // key dispatch; the same activity will pick it up again on wakeup.
                if (!uiSleeping) {
                    prev.pauseKeyDispatchingLocked();
                } else {
                    if (DEBUG_PAUSE) Slog.v(TAG, "Key dispatch not paused for screen off");
                }

                if (dontWait) {
                    // If the caller said they don't want to wait for the pause, then complete
                    // the pause now.
                    completePauseLocked(false);
                    return false;

                } else {
                    // Schedule a pause timeout in case the app doesn't respond.
                    // We don't give it much time because this directly impacts the
                    // responsiveness seen by the user.
                    Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
                    msg.obj = prev;
                    prev.pauseTime = SystemClock.uptimeMillis();
                    mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
                    if (DEBUG_PAUSE) Slog.v(TAG, "Waiting for pause to complete...");
                    return true;
                }

            } else {
                // This activity failed to schedule the
                // pause, so just treat it as being paused now.
                if (DEBUG_PAUSE) Slog.v(TAG, "Activity not running, resuming next.");
                if (!resuming) {
                    mStackSupervisor.getFocusedStack().resumeTopActivityLocked(null);
                }
                return false;
            }
        }

    9、ActivityThread的schedulePauseActivity方法

        public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
        }

        case PAUSE_ACTIVITY:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                    handlePauseActivity((IBinder)msg.obj, false, (msg.arg1&1) != 0, msg.arg2,
                            (msg.arg1&2) != 0);
                    maybeSnapshot();
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;

        private void handlePauseActivity(IBinder token, boolean finished,
            boolean userLeaving, int configChanges, boolean dontReport) {
            ActivityClientRecord r = mActivities.get(token);
            if (r != null) {
                //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
                if (userLeaving) {
                    performUserLeavingActivity(r);
                }

                r.activity.mConfigChangeFlags |= configChanges;
                performPauseActivity(token, finished, r.isPreHoneycomb());

                // Make sure any pending writes are now committed.
                if (r.isPreHoneycomb()) {
                    QueuedWork.waitToFinish();
                }

                //告诉activity，我们已经暂停了
                // Tell the activity manager we have paused.
                if (!dontReport) {
                    try {
                        ActivityManagerNative.getDefault().activityPaused(token);
                    } catch (RemoteException ex) {
                    }
                }
                mSomeActivitiesChanged = true;
            }
        }

        performPauseActivity方法，因为r不为null，所以执行下面

        final Bundle performPauseActivity(IBinder token, boolean finished,
            boolean saveState) {
            ActivityClientRecord r = mActivities.get(token);
            return r != null ? performPauseActivity(r, finished, saveState) : null;
        }

        final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
                boolean saveState) {
            。。。

            try {
                //保存当前状态
                // Next have the activity save its current state and managed dialogs...
                if (!r.activity.mFinished && saveState) {
                    callCallActivityOnSaveInstanceState(r);
                }
                // Now we are idle.
                r.activity.mCalled = false;
                //回调到onPause方法
                mInstrumentation.callActivityOnPause(r.activity);
                。。。

        }

        至此，我们已经把之前的activity暂停了，那是怎么继续生命周期里的onResume呢？我们来看暂停后的代码

                // Tell the activity manager we have paused.
                if (!dontReport) {
                    try {
                        ActivityManagerNative.getDefault().activityPaused(token);
                    } catch (RemoteException ex) {
                    }
                }
                这段代码通过进程间通信，AMS告诉activity，我们已经停止之前的activity了，你可以继续处理了

    10、现在我们来看下AMS里的activityPause方法

        @Override
        public final void activityPaused(IBinder token) {
            final long origId = Binder.clearCallingIdentity();
            synchronized(this) {
                ActivityStack stack = ActivityRecord.getStackLocked(token);
                if (stack != null) {
                    stack.activityPausedLocked(token, false);
                }
            }
            Binder.restoreCallingIdentity(origId);
        }

        这里面只是拿到了这个的ActivityStack，调用了它的activityPausedLocked方法

    11、现在来看下activityPausedLocked方法

        final void activityPausedLocked(IBinder token, boolean timeout) {
            if (DEBUG_PAUSE) Slog.v(
                TAG, "Activity paused: token=" + token + ", timeout=" + timeout);

            final ActivityRecord r = isInStackLocked(token);
            if (r != null) {
                mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
                if (mPausingActivity == r) {
                    if (DEBUG_STATES) Slog.v(TAG, "Moving to PAUSED: " + r
                            + (timeout ? " (due to timeout)" : " (pause complete)"));
                    completePauseLocked(true);
                } else {
                    。。。
            }
        }
        我们知道之前给mPasingActivity赋值了，所有拿到的是相等的，执行下面的


        private void completePauseLocked(boolean resumeNext) {
            ActivityRecord prev = mPausingActivity;
            //prev不为null，所以执行下面的
            if (prev != null) {
                prev.state = ActivityState.PAUSED;
                
                ...

                //把mPausingActivity设为null
                mPausingActivity = null;
            }

            if (resumeNext) {
                final ActivityStack topStack = mStackSupervisor.getFocusedStack();
                if (!mService.isSleepingOrShuttingDown()) {
                    mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
                } else {
                    mStackSupervisor.checkReadyForSleepLocked();
                    ActivityRecord top = topStack.topRunningActivityLocked(null);
                    if (top == null || (prev != null && top != prev)) {
                        // If there are no more activities available to run,
                        // do resume anyway to start something.  Also if the top
                        // activity on the stack is not the just paused activity,
                        // we need to go ahead and resume it to ensure we complete
                        // an in-flight app switch.
                        mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
                    }
                }
            }

            。。。   
        }    

        下面有跳转到了ActivityStackSupervison的resumeTopActivitiesLocked方法了 

    12、 ActivityStackSupervison的resumeTopActivitiesLocked方法

        同上面第7，8步，这次mResumeActivity是null，所以不执行onPause，next是我们要启动的activity。
        如果是Launcher启动，则next。app是null，如果不是，怎next。app不为null


    13、ActivityStackSupervisor的startSpecificActivityLocked方法

        void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
            // Is this activity's application already running?
            ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                    r.info.applicationInfo.uid, true);

            r.task.stack.setLaunchTime(r);

            //A启动B，则是有app和app.thread的，如果是首次启动，则是没有的，不执行if语句
            if (app != null && app.thread != null) {
                try {
                    if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                            || !"android".equals(r.info.packageName)) {
                        // Don't add this if it is a platform component that is marked
                        // to run in multiple processes, because this is actually
                        // part of the framework so doesn't make sense to track as a
                        // separate apk in the process.
                        app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                                mService.mProcessStats);
                    }
                    realStartActivityLocked(r, app, andResume, checkConfig);
                    return;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting activity "
                            + r.intent.getComponent().flattenToShortString(), e);
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }

            //如果目标进程还没有启动，则不会执行上面if语句，如果是Launcher,则会创建
            mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                    "activity", r.intent.getComponent(), false, false, true);
        }

        我们看到通过AMS拿到app的ProcessRecord,可以看下代码

        private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        
            。。。

            
                Process.ProcessStartResult startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, debugFlags, mountExternal,
                        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                        app.info.dataDir, entryPointArgs);
                
                
               。。。
        }
        从这我们可以知道，这是启动了一个新的进程来启动app，只有首次启动时才有，执行完这一步后，就进入ActivityThread的main方法里了
        由于main方法里new了个ActivityThread并调用了attach方法，attach方法里，调用了AMS的attachApplication把mainThread作为参数传进去了，我们看下AMS的attachApplication方法可以知道，里面调用了attachApplicationLocked方法，这个方法里最终调用了ealStartActivityLocked方法

        我们先看realStartActivityLocked方法的具体实现
        final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

            。。。

                ProfilerInfo profilerInfo = profileFile != null
                        ? new ProfilerInfo(profileFile, profileFd, mService.mSamplingInterval,
                        mService.mAutoStopProfiler) : null;
                app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_TOP);
                app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                        System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                        r.compat, r.task.voiceInteractor, app.repProcState, r.icicle, r.persistentState,
                        results, newIntents, !andResume, mService.isNextTransitionForward(),
                        profilerInfo);

                。。。
        }

        最后我们调用了app.thread也就是ui线程（ActivityTread）的scheduleLaunchActivity方法，这其中还是涉及到了进程间通信，下面我们来看下ui线程是怎么处理创建的

    14、ui线程的scheduleLaunchActivity方法

        // we use token to identify this activity without having to send the
        // activity itself back to the activity manager. (matters more with ipc)
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
                IVoiceInteractor voiceInteractor, int procState, Bundle state,
                PersistableBundle persistentState, List<ResultInfo> pendingResults,
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward,
                ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
        }

        我们可以看到，最后发送了一个message，这就是我们常用的Handle，Message等，我们下面看下怎么处理的

        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;


        private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
            。。。

            Activity a = performLaunchActivity(r, customIntent);

            if (a != null) {
                r.createdConfig = new Configuration(mConfiguration);
                Bundle oldState = r.state;
                handleResumeActivity(r.token, false, r.isForward,
                        !r.activity.mFinished && !r.startsNotResumed);

                if (!r.activity.mFinished && r.startsNotResumed) {
                    。。。

                    r.paused = true;
                }
            } else {
                //有错误了话，就调用了finish了
                // If there was an error, for any reason, tell the activity
                // manager to stop us.
                try {
                    ActivityManagerNative.getDefault()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
                } catch (RemoteException ex) {
                    // Ignore
                }
            }
        }

        performLaunchActivity方法生成了一个activity，我们看下怎么生成的

        private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
            // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");

            //先拿到activityInfo信息
            ActivityInfo aInfo = r.activityInfo;
            if (r.packageInfo == null) {
                r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                        Context.CONTEXT_INCLUDE_CODE);
            }

            ComponentName component = r.intent.getComponent();
            if (component == null) {
                component = r.intent.resolveActivity(
                    mInitialApplication.getPackageManager());
                r.intent.setComponent(component);
            }

            if (r.activityInfo.targetActivity != null) {
                component = new ComponentName(r.activityInfo.packageName,
                        r.activityInfo.targetActivity);
            }

            Activity activity = null;
            try {
                //通过反射生成activity
                java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
                activity = mInstrumentation.newActivity(
                        cl, component.getClassName(), r.intent);
                StrictMode.incrementExpectedActivityCount(activity.getClass());
                r.intent.setExtrasClassLoader(cl);
                r.intent.prepareToEnterProcess();
                if (r.state != null) {
                    r.state.setClassLoader(cl);
                }
            } catch (Exception e) {
                if (!mInstrumentation.onException(activity, e)) {
                    throw new RuntimeException(
                        "Unable to instantiate activity " + component
                        + ": " + e.toString(), e);
                }
            }

            try {
                Application app = r.packageInfo.makeApplication(false, mInstrumentation);

                if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
                if (localLOGV) Slog.v(
                        TAG, r + ": app=" + app
                        + ", appName=" + app.getPackageName()
                        + ", pkg=" + r.packageInfo.getPackageName()
                        + ", comp=" + r.intent.getComponent().toShortString()
                        + ", dir=" + r.packageInfo.getAppDir());

                if (activity != null) {
                    Context appContext = createBaseContextForActivity(r, activity);
                    CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                    Configuration config = new Configuration(mCompatConfiguration);
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                            + r.activityInfo.name + " with config " + config);
                    activity.attach(appContext, this, getInstrumentation(), r.token,
                            r.ident, app, r.intent, r.activityInfo, title, r.parent,
                            r.embeddedID, r.lastNonConfigurationInstances, config,
                            r.voiceInteractor);

                    if (customIntent != null) {
                        activity.mIntent = customIntent;
                    }
                    r.lastNonConfigurationInstances = null;
                    activity.mStartedActivity = false;
                    int theme = r.activityInfo.getThemeResource();
                    if (theme != 0) {
                        activity.setTheme(theme);
                    }

                    activity.mCalled = false;

                    //回调到activity的onCreate方法
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnCreate(activity, r.state);
                    }
                    。。。

                    r.activity = activity;
                    r.stopped = true;
                    //回调activity的onStart方法
                    if (!r.activity.mFinished) {
                        activity.performStart();
                        r.stopped = false;
                    }
                    //
                    if (!r.activity.mFinished) {
                        if (r.isPersistable()) {
                            if (r.state != null || r.persistentState != null) {
                                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                        r.persistentState);
                            }
                        } else if (r.state != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                        }
                    }
                    //onPostCreate方法
                    if (!r.activity.mFinished) {
                        activity.mCalled = false;
                        if (r.isPersistable()) {
                            mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                    r.persistentState);
                        } else {
                            mInstrumentation.callActivityOnPostCreate(activity, r.state);
                        }
                        if (!activity.mCalled) {
                            throw new SuperNotCalledException(
                                "Activity " + r.intent.getComponent().toShortString() +
                                " did not call through to super.onPostCreate()");
                        }
                    }
                }
                r.paused = true;

                mActivities.put(r.token, r);

            。。。

            return activity;
        }

        至此，我们已经知道了activity的onCreate,onStart流程了,那是怎么调用onResume的呢？我们来看创建activity后的代码
        判断activity不为null后，直接调用handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed);方法，我们来看看这个的代码

        final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
            
            ...

            // TODO Push resumeArgs into the activity for consideration
            ActivityClientRecord r = performResumeActivity(token, clearHide);

            if (r != null) {
                final Activity a = r.activity;

                ...

                //用来显示ui的，包括测量，布局，绘制
                if (r.window == null && !a.mFinished && willBeVisible) {
                    r.window = r.activity.getWindow();
                    View decor = r.window.getDecorView();
                    decor.setVisibility(View.INVISIBLE);
                    ViewManager wm = a.getWindowManager();
                    WindowManager.LayoutParams l = r.window.getAttributes();
                    a.mDecor = decor;
                    l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                    l.softInputMode |= forwardBit;
                    if (a.mVisibleFromClient) {
                        a.mWindowAdded = true;
                        wm.addView(decor, l);
                    }

                ...

                if (!r.onlyLocalRequest) {
                    r.nextIdle = mNewActivities;
                    mNewActivities = r;
                    if (localLOGV) Slog.v(
                        TAG, "Scheduling idle handler for " + r);
                    //这是调用之前activity的onStop的
                    Looper.myQueue().addIdleHandler(new Idler());

                ...
            }
        }


        public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide) {
            ActivityClientRecord r = mActivities.get(token);
            if (localLOGV) Slog.v(TAG, "Performing resume of " + r
                    + " finished=" + r.activity.mFinished);
            if (r != null && !r.activity.mFinished) {
                。。。
                    r.activity.performResume();

                    EventLog.writeEvent(LOG_ON_RESUME_CALLED,
                            UserHandle.myUserId(), r.activity.getComponentName().getClassName());

                    r.paused = false;
                    r.stopped = false;
                    r.state = null;
                    r.persistentState = null;
                。。。
            return r;
        }

        执行到了activity的performResume方法

    15、Activity的performResume方法

        final void performResume() {
            //restart方法
            performRestart();

            mFragments.execPendingActions();

            mLastNonConfigurationInstances = null;

            mCalled = false;
            //回调到了onResume方法
            // mResumed is set by the instrumentation
            mInstrumentation.callActivityOnResume(this);
            if (!mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onResume()");
            }

            // Now really resume, and install the current status bar and menu.
            mCalled = false;

            mFragments.dispatchResume();
            mFragments.execPendingActions();

            onPostResume();
            if (!mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onPostResume()");
            }
        }


        final void performRestart() {
            mFragments.noteStateNotSaved();

            if (mStopped) {
                mStopped = false;
                。。。

                mCalled = false;
                //回调到onRestart方法
                mInstrumentation.callActivityOnRestart(this);
                if (!mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + mComponent.toShortString() +
                        " did not call through to super.onRestart()");
                }
                //回调到onStart方法
                performStart();
            }
        }


    16、我们来看下如果是首次启动的情况下，怎么创建线程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                    "activity", r.intent.getComponent(), false, false, true);


        final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
            ..
            startProcessLocked(
                    app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
            checkTime(startTime, "startProcess: done starting proc!");
            return (app.pid != 0) ? app : null;
        }


        private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
            ...
                Process.ProcessStartResult startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, debugFlags, mountExternal,
                        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                        app.info.dataDir, entryPointArgs);
                ...
        }



    17、Looper.myQueue().addIdleHandler(new Idler());调用到上一个activity的onStop方法里
            if (!r.onlyLocalRequest) {
                r.nextIdle = mNewActivities;
                mNewActivities = r;
                if (localLOGV) Slog.v(
                    TAG, "Scheduling idle handler for " + r);
                Looper.myQueue().addIdleHandler(new Idler());
            }

        private class Idler implements MessageQueue.IdleHandler {
            @Override
            public final boolean queueIdle() {
                ActivityClientRecord a = mNewActivities;
                boolean stopProfiling = false;
                if (mBoundApplication != null && mProfiler.profileFd != null
                        && mProfiler.autoStopProfiler) {
                    stopProfiling = true;
                }
                if (a != null) {
                    mNewActivities = null;
                    IActivityManager am = ActivityManagerNative.getDefault();
                    ActivityClientRecord prev;
                    do {
                        if (localLOGV) Slog.v(
                            TAG, "Reporting idle of " + a +
                            " finished=" +
                            (a.activity != null && a.activity.mFinished));
                        if (a.activity != null && !a.activity.mFinished) {
                            try {
                                am.activityIdle(a.token, a.createdConfig, stopProfiling);
                                a.createdConfig = null;
                            } catch (RemoteException ex) {
                                // Ignore
                            }
                        }
                        prev = a;
                        a = a.nextIdle;
                        prev.nextIdle = null;
                    } while (a != null);
                }
                if (stopProfiling) {
                    mProfiler.stopProfiling();
                }
                ensureJitEnabled();
                return false;
            }
        }


    18、AMS的activityIdle方法

        @Override
        public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
            final long origId = Binder.clearCallingIdentity();
            synchronized (this) {
                ActivityStack stack = ActivityRecord.getStackLocked(token);
                if (stack != null) {
                    ActivityRecord r =
                            mStackSupervisor.activityIdleInternalLocked(token, false, config);
                    if (stopProfiling) {
                        if ((mProfileProc == r.app) && (mProfileFd != null)) {
                            try {
                                mProfileFd.close();
                            } catch (IOException e) {
                            }
                            clearProfilerLocked();
                        }
                    }
                }
            }
            Binder.restoreCallingIdentity(origId);
        }

    19、ActivityStackSupervisor的ctivityIdleInternalLocked方法

        final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
                Configuration config) {

            。。。

            // Atomically retrieve all of the other things to do.
            stops = processStoppingActivitiesLocked(true);
            NS = stops != null ? stops.size() : 0;
            if ((NF=mFinishingActivities.size()) > 0) {
                finishes = new ArrayList<ActivityRecord>(mFinishingActivities);
                mFinishingActivities.clear();
            }

            if (mStartingUsers.size() > 0) {
                startingUsers = new ArrayList<UserStartedState>(mStartingUsers);
                mStartingUsers.clear();
            }

            // Stop any activities that are scheduled to do so but have been
            // waiting for the next one to start.
            for (int i = 0; i < NS; i++) {
                r = stops.get(i);
                final ActivityStack stack = r.task.stack;
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
                } else {
                    //调用到这里
                    stack.stopActivityLocked(r);
                }
            }

            。。。

            return r;
        }

    20、下面我们来看下ActivityStack里的stopActivityLocked方法

        final void stopActivityLocked(ActivityRecord r) {
            。。。

            //r是已经存在的，所以if判断是true
            if (r.app != null && r.app.thread != null) {
                adjustFocusedActivityLocked(r);
                r.resumeKeyDispatchingLocked();
                try {
                    r.stopped = false;
                    if (DEBUG_STATES) Slog.v(TAG, "Moving to STOPPING: " + r
                            + " (stop requested)");
                    r.state = ActivityState.STOPPING;
                    if (DEBUG_VISBILITY) Slog.v(
                            TAG, "Stopping visible=" + r.visible + " for " + r);
                    //设置visible成false
                    if (!r.visible) {
                        mWindowManager.setAppVisibility(r.appToken, false);
                    }
                    //调用ActivityThread的scheduleStopActivity方法
                    r.app.thread.scheduleStopActivity(r.appToken, r.visible, r.configChangeFlags);
                    。。。
                } catch (Exception e) {
                    。。。
                }
            }
        }

    21、下面调用到了ActivityThread的scheduleStopActivity方法

        public final void scheduleStopActivity(IBinder token, boolean showWindow,
                int configChanges) {
           sendMessage(
                showWindow ? H.STOP_ACTIVITY_SHOW : H.STOP_ACTIVITY_HIDE,
                token, 0, configChanges);
        }

        下面经过消息队列处理，到handleStopActivity方法

        private void handleStopActivity(IBinder token, boolean show, int configChanges) {
            ActivityClientRecord r = mActivities.get(token);
            r.activity.mConfigChangeFlags |= configChanges;

            StopInfo info = new StopInfo();
            performStopActivityInner(r, info, show, true);

            。。。
        }

        private void performStopActivityInner(ActivityClientRecord r,
            StopInfo info, boolean keepShown, boolean saveState) {
            if (localLOGV) Slog.v(TAG, "Performing stop of " + r);
            if (r != null) {
                。。。

                // Next have the activity save its current state and managed dialogs...
                if (!r.activity.mFinished && saveState) {
                    if (r.state == null) {
                        callCallActivityOnSaveInstanceState(r);
                    }
                }

                if (!keepShown) {
                    try {
                        // Now we are idle.
                        r.activity.performStop();
                    } catch (Exception e) {
                        if (!mInstrumentation.onException(r.activity, e)) {
                            throw new RuntimeException(
                                    "Unable to stop activity "
                                    + r.intent.getComponent().toShortString()
                                    + ": " + e.toString(), e);
                        }
                    }
                    r.stopped = true;
                }

                r.paused = true;
            }
        }

        可以看到，最后才把r.paused状态设置为true，所以之前都是false的状态的

    22、调用到了Activity的performStop方法，我们来看看里面进行了怎样的处理

        final void performStop() {
        
            。。。

            if (!mStopped) {
                ...
                //调回onStop方法
                mInstrumentation.callActivityOnStop(this);
                if (!mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + mComponent.toShortString() +
                        " did not call through to super.onStop()");
                }

                synchronized (mManagedCursors) {
                    final int N = mManagedCursors.size();
                    for (int i=0; i<N; i++) {
                        ManagedCursor mc = mManagedCursors.get(i);
                        if (!mc.mReleased) {
                            mc.mCursor.deactivate();
                            mc.mReleased = true;
                        }
                    }
                }

                mStopped = true;
            }
            mResumed = false;
        }


    最后我们说下，如果不是启动一个activity，而是finish一个activity，这个流程是怎样的呢
    会调用finish的销毁流程，会接着调用之前的流程，可以自己研究下

    我们研究的总流程来做个总结，A启动B，如果是首次启动，也差不多一样的流程

    B.onPause -> A.onCreate -> A.onStart -> A.onResume -> B.onStop




    




	