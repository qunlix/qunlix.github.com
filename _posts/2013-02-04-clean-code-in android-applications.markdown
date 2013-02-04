---
layout: post
title: CLEAN CODE IN ANDROID APPLICATIONS
categories:
- android 
---
Let's say you wake up one morning, and think, "Hey, I'm going to build an Android app today." First off, good choice! As of the end of June, 500,000 Android devices were being activated every day, outpacing even the iPhone. That means there is a large, potential audience for your app. Additionally, Android is built with Java. This may not seem like a big deal, but I have worked in Objective-C on the iOS platform for a few years, and while I am now quite comfortable with it, the iOS SDK offered a steeper learning curve than I experienced with Android. Android just felt more accessible when I first started working with the Android SDK. That said, there are some clear differences from any other Java application you have built in the past, and I'll go over some of those in the first section.
So moving forward in time, you have completed your first app, and have submitted it to the Android Market. Congratulations are in order, as your friends are all downloading your app and tweeting about it. Now it is time to start on your second app. You spend a few days, and suddenly realize that you are starting to reuse code from your first app, which in itself is not a bad thing. Code reuse can be valuable. But you notice there is a lot of boilerplate code that tends to be repeated often, and that can be distracting from focusing on your business logic. Fortunately, there are some ways to improve upon this.
In this blog post, I will provide an overview of Android and the application lifecycle, and discuss some of the limitations imposed by the framework. I will also review a few of the techniques and third party projects that can help you clean up your Android code, and focus on what you want to achieve with your app.
## Android Overview ##
Let's begin with a brief overview of how Android works. Android applications (apps) are built using Java, and compiled to class files. The class files are then compiled into the Dalvik Executable (DEX) format, so they may run on the Dalvik virtual machine used by Android. After conversion to DEX format, the class files are zipped to an Android Package (APK) for distribution to devices. Because of the use of the DEX format, the Dalvik VM is not a true Java Virtual Machine, since it does not operate on Java byte code. Additionally, the Dalvik VM is based on a subset of the Apache Harmony project for its core class library. This means that many of the classes and methods to which you are accustomed in Java SE are available, but certainly not all. I have found the API reference on the Android developer web site to be an invaluable resource for reviewing these differences.
By default, each Android application is assigned a unique Linux user ID by the Android operating system. When started by the system, an application runs in its own Linux process, within its own virtual machine (VM). The system manages the starting and shutting down of this process when needed. As you can guess, this means that each application runs in isolation from the other running applications. When installed, an app can request permission to access hardware features or interact with other applications. The user elects to grant these permissions to the app or to not install it. The permissions required or requested by an app are defined in each app's Android Manifest file. This is an XML file that lists all the components of the app, and any settings for those components. The four types of application components are activities, services, content providers, and broadcast receivers. For the purposes of this post, I will be focusing on activities.
Activities basically represent a single screen of an Android application. For example, a Twitter app may have a login screen, a screen with a list of tweets, and a screen for authoring a new tweet. Each of these screens represent different activities within the application. As a developer you never instantiate an activity object yourself. Activities are activated by sending an asynchronous message called an Intent, as seen in the example below.

`startActivity(new Intent(context, HomeActivity.class));`

When startActivity(Intent intent) is called, the system either creates a new instance or reuses an existing one in order to display the activity to the user. The important point is that the system controls the starting and stopping, and creation and destroying of the application and each activity. If you want to interact with this process, then the application and activity classes provide methods for different lifecycle events that you can override in a subclass.

## Dependency Injection ##
The Spring Android project recently reached its fourth milestone release. With that release we have continued to improve upon the RestTemplate and Spring Social support for Android, which simplifies the process of making RESTful HTTP requests and accessing REST APIs secured by OAuth. And while we believe these are valuable additions for Android development, some developers have asked questions regarding the possibility of dependency injection support in Spring Android, because as you are probably aware, the Spring Framework already provides a popular Inversion of Control (IOC) container for enabling dependency injection in enterprise Java applications. Early in the Spring Android planning stages, dependency injection support was identified as a potential candidate for inclusion in the project. At that point, it was unclear what that support would entail, and how it would be implemented. Because of this, I began the process of researching and investigating the possible methods available for, and limitations of, performing dependency injection in Android.
Well, what is dependency injection? If you ask two different developers, you may get two different answers. You might hear about IOC, XML files, annotations, or some other implementation detail. In reality, dependency injection is simply a technique to reduce coupling by handing an object what it needs to work, rather than having the object reach out into its environment. That sounds easy enough, and you might be thinking to yourself you can already get this with class constructors and setter methods, which is completely true. However, recall from the overview section above, the Android system drives the application lifecycle, so the way we can do this is limited.
## The Android Way ##
Without using any third party libraries, it is rather easy to pass a dependency to an Activity. As discussed earlier, the system creates the application instance. So by extending application, you can effectively create a singleton dependency instance, which can then be accessed by any of the activities in the app.

`public class MainApplication extends Application  {
 
    private MyService service;
 
    @Override
    public void onCreate() {
        super.onCreate();
        service = new MyServiceImpl();
    }
 
    public MyService getMyService() {
        return this.service;
    }
}`


The activity class has a method called getApplication() which returns a reference to the application object that owns the activity. We simply cast it to MainApplication, and we can access the getter method for the MyService. Of course, the activity now has to "know" about the application, which might seem like a disadvantage. But remember, the activity already knows about its application. The method is built in.

{% highlight ruby %} public class MainActivity extends Activity  {
 
    private MyService service;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MainApplication app = (MainApplication) getApplication();
        service = app.getMyService();
    }
}{% endhighlight  %} 

## RoboGuice ##
The RoboGuice project utilizes Google's Guice library to add dependency injection support to Android. Guice itself comes in two flavors, with and without AOP (Aspect Oriented Programming) support. Internally, standard Guice relies on bytecode generation to perform method interception. Android however, does not support runtime bytecode generation, so RoboGuice depends on the version of Guice without AOP. Let's look at how we would implement the previous example using RoboGuice. To add a custom binding, you must implement an Application object that extends from RoboApplication. You then override the addApplicationModules(…) method, and add a module instance that binds your objects.

`
public class MainApplication extends RoboApplication {
 
    @Override
    protected void addApplicationModules(List<Module> modules) {
        // add your module with custom bindings
        modules.add(new MainModule());
    }
 
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
`

AbstractAndroidModule extends from the the standard Guice AbstractModule. Override the configure() method to specify bindings:

`
public class MainModule extends AbstractAndroidModule {
 
    @Override
    protected void configure() {
        bind(MyService.class).to(MyServiceImpl.class);
    }
}
`

Each activity must inherit from RoboActivity so that injection can occur:

`
public class MainActivity extends RoboActivity {
 
    @Inject
    MyService service;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}
`

This example is not overly impressive, since it requires more code to accomplish a similar task. RoboGuice wiring becomes more useful in larger applications that consist of multiple domain modules encapsulating considerable business logic. Also, the support it provides for injecting views, resources, and system services is generally useful. You can see this in the examples below. The first example illustrates the standard android approach, while the second utilizes RoboGuice. As you can see, RoboGuice allows you to eliminate much of the boilerplate lookup code.

`
public class MyActivity extends Activity {
 
    private TextView label;
 
    private Drawable image;
 
    private SearchManager searchManager;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.myactivity);
        this.label = (TextView) findViewById(R.id.mylabel);
        this.image = getResources().getDrawable(R.drawable.myimage);
        this.searchManager = (SearchManager) getSystemService(Activity.SEARCH_SERVICE);
    }
}
`


`
public class MyActivity extends RoboActivity {
 
    @InjectView(R.id.mylabel)
    TextView label;
 
    @InjectResource(R.drawable.myimage)
    Drawable image;
 
    @Inject
    SearchManager searchManager;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.myactivity);
    }
}
`
Some additional things about the programming model jump out at me here. Previously we extended from standard android classes, now we extend from RoboGuice specific variants. This takes us back to the previous section about the architecture of Android. Subclassing framework specific types is required because the way a third-party framework can hook into the Android application lifecycle is limited. In addition, injection is performed at runtime which brings a performance and footprint cost. Finally, it is worth noting the latest released version is built on Guice 2.0. My understanding is an update to Guice 3.0 is in progress.
I am currently undecided about whether the benefits of RoboGuice are great enough to warrant tightly coupling your application with a third party framework. I have generally found it good enough to implement my own manual dependency injection techniques, as outlined in the previous section and demonstrated in the Greenhouse reference application. Additionally, Guice itself is not small, adding about 400 KB to the size of your application. The RoboGuice project page even discusses this in reference to configuring ProGuard to reduce the size of your app. While configuring ProGuard is not necessarily trivial, it is also a good practice to use it on any app, regardless of whether you are using RoboGuice or not.
## Beyond Dependency Injection ##
We've covered dependency injection and RoboGuice, and discussed the advantages and disadvantages of that project's approach. The title of this post is about clean code in Android and how to reduce redundancy, so let's now step beyond dependency injection, and discuss some other techniques.
## Android Annotations ##
Android Annotations is a project started and maintained by Pierre-Yves Ricau. Through the use of annotations, the goal of the project is specifically to help reduce the amount of boilerplate code in Android projects. It's not trying to be a general dependency injection framework, and in fact, can work side by side with RoboGuice. There is some overlap, as Android Annotations provides some similar annotations for injecting views and resources, but it offers many other useful features as well. The main differentiator is that Android Annotations generates the boilerplate code at compile time, so there is no runtime penalty for using it. It does this by generating a subclass of each activity, and substituting the annotations with the standard boilerplate code. One minor disadvantage to this approach is that you must append an underscore to the name of each activity in your manifest file. For example, if I create a MyActivity class, Android Annotations will generate a corresponding MyActivity_ class. Additionally, any reference in startActivity(…) has to be updated to use the new class as well.
Here you can see some similarities to RoboGuice. This is the Android Annotations version of the previous RoboGuice example. Note that using the @EActivity annotation on the class allows you to set the activity's layout. Another nice touch is that if you omit the resource id from the annotation, Android Annotations will look for a resource id that matches the name of the variable.

`@EActivity(R.layout.myactivity) // Sets content view to R.layout.myactivity
public class MyActivity extends Activity {
 
    @InjectView  // Injects R.id.mylabel
    TextView mylabel;
 
    @DrawableRes(R.drawable.myimage)
    Drawable image;
 
    @SystemService
    SearchManager searchManager;
}`

The annotations library is fairly extensive. Currently, it supports several areas of the SDK:
- views
- event handling
- accessing resources
- extras
- system services
- threading
Here are a few more examples of what is possible. I want to highlight the click event handing and threading support, because they really illustrate the power of the library.

`public class AnotherActivity extends Activity {
 
    @Click // When R.id.runProcess button is clicked
    void runProcess() {
        runInBackground();
    }
 
   @Background // Executed in a background thread
   void runInBackground() {
        // perform some long running task
        notifyUser();
    }
 
    @UiThread // Executed in the ui thread
    void notifyUser() {
         // display a notification that task is complete
    }
}`

Some developers may be turned off by the use of code generation, but considering the limitations of the Android architecture, Android Annotations provides an elegant approach to minimizing the use of boilerplate code, helping you focus on your business logic while not increasing runtime footprint.
## Android Binding ##
The final project I want to touch on is Android Binding. Android Binding is a MVVM (Model-View-ViewModel) framework, designed to separate the activity from working directly on the user interfaces. To do this, almost all of the code is moved out of the Android activity class and placed in a ViewModel object, which is meant to help with testability. The ViewModel object handles all the events and data for the view (the layout), but it is not tightly coupled to it. Events that occur in the view are sent to the ViewModel as commands. These commands then execute some business logic and possibly interact with the model, and update the ViewModel's properties so the View can then bind to them. If you have ever worked with .Net, then the concept of data binding will be quite familiar to you, and Android Binding strives to apply that concept to Android. Android Binding was started and is being maintained by Andy Tsui. It is still young and lacks documentation, but there is some nice potential.
The example below shows an EmailActivity extending from BindingActivity. When you extend from BindingActivity, you use the setAndBindRootView(…) method to set the Model for the view:

`public class EmailActivity extends BindingActivity {
 
    private EmailViewModel model;
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = new EmailViewModel();
        setAndBindRootView(R.layout.main, model);
    }
}`

The next example snippet shows the view declaration. For me, this is the really interesting part. Looking at this layout file, you can see the addition of the "binding" attribute, as well as a custom "binding" xmlns declaration. This example shows binding two text fields and a button, but the project supports binding to many other controls..

`<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:binding="http://www.gueei.com/android-binding"
    android:orientation="vertical"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent">
 
    <EditText
        android:id="@+id/subject"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        binding:text="subject"
    />
 
    <EditText
        android:id="@+id/message"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        binding:text="message"
    />
 
    <Button
        android:id="@+id/submit"
        android:text="Submit"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        binding:onClick="submit"
    />
 
</LinearLayout>`

Lastly, the EmailViewModel contains the data and control logic for the view. The excerpt below shows the Observables for the textfield values, as well as the Command that executes when the submit button is clicked:

`public class EmailViewModel {
 
    @Required
    public StringObservable subject = new StringObservable();
 
    @Required
    public StringObservable message = new StringObservable();
 
    public Command submit = new Command() {
        public void Invoke(View arg0, Object... arg1) {
            ValidationResult result = ModelValidator.ValidateModel(EmailViewModel.this);
            if (result.isValid()){
                // send email
            } else {
                // display validation error message
            }
        }
    };
}`

There is more to this library than what I've included here. It also includes support for model validation through the use of a set of annotations. There are some built in rules, and you can use RegEx pattern matching, as well as implement your own custom rules. The recent version has also added support for binding to Options menus.
Like RoboGuice, Android Binding is runtime-based and requires you to extend from specific Activity base classes. Such reliance on concrete inheritance would likely make it difficult to use RoboGuice and Binding together in the same application. Where I think Binding really shines is in larger business applications that have data entry and validation requirements. The declarative xml binding syntax is pretty nice. I'm not familiar with other projects doing this, and it shows a fresh perspective on Android development.
## Closing Thoughts ##
All of these techniques and projects are striving to implement known design patterns within the Android framework. This is a worthy goal, because when applied correctly, these patterns can help us better structure our applications. Generally speaking, when I approach a new platform or language, I try to work with what the SDK provides first. From there I can decide whether to use a third party library or apply a specific design pattern. One of my concerns is that sometimes design patterns or ideas that work well in one situation or technology do not translate well to another one. In spite of the limitations of the Android architecture, the techniques and projects highlighted in this post succeed in helping you write cleaner code.
Of all the approaches I have discussed in this post, I am most interested in Android Annotations. I like it because it is designed to keep your code clean and readable, without sacrificing performance or using additional resources. Because it generates standard Android code at compile time, this means you can debug your app without having to step through a third party library. Spring Roo takes a very similar approach, except its code generation model is based on AspectJ instead of Java Annotation processing. It would be exciting to see if the Android Annotations project could be integrated into a Spring Roo add-on. This would allow the community to create new Roo commands to scaffold Android projects and types, among other things. In addition, since Roo uses AspectJ compile-time weaving, it would be possible to eliminate the need for separate classes generated with the underscore naming.
In conclusion, I have illustrated the use of some techniques and third party libraries to help eliminate boilerplate code and simplify the development process of an Android app. Yes, there is some setup involved in preparing your environment. However, the benefits can outweigh that initial time investment. I recommend devoting some time to each of these projects and techniques. They each offer a unique and valuable perspective on Android development.
Thanks for reading! I would really enjoy hearing your feedback. What do you think of the libraries discussed in this article – are you using them? Are there other libraries you consider essential? Would you find value in a Roo add-on for Android that builds on the Android Annotations project?