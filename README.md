# Java Style Guide

Heavily influenced by Android's [Code Style for Contributors](https://source.android.com/source/code-style.html). Many thanks to the editors of this detailed and thoughtful guide.

## Environment

Before you get to writing, make sure you're set up for the right kind of code.

### Android Studio

Android Studio does a lot of sensible things. Don't go out of your way to fight it. The exceptional times when it is a bother are worth discussing as a team to document.

* Gradle is our build tool and dependency manager.
* Objects should stored in single-depth subfolders with reasonable names like "activities", "fragments", "models", etc.
* Colors and numbers should be abstracted into their own respective values XML files.
* Target the latest API and add support backwards as needed. Usually support libraries are used but don't assume they'll be needed when you start a new project.

## User Interface

Code is meaningless if there's no way for the user to use it. The UI programming should leverage as many tools as are available.

### Layouts

Keep your layout XML as simple as possible. Don't bind click events or any other logic. If a view has conditions, make separate views that are composable and load the appropriate one as needed.

Avoid writing layouts in code. There's nothing inherently wrong with code-based layouts but it helps to separate concerns, allow non-developers easier access to layout modifications, and generally keep classes smaller sizes.

### Gestures

Remember you're developing for a mobile device so interactions with elements are different than would be on a keyboard- and mouse-based system. Users should be able to tap, swipe, pinch, and otherwise navigate the UI using their hands. 

Related, avoid designs that are overly busy, nested, cramped, or would cause performance issues. It's usually better to fall back to more simple ideas that users will intuitively understand than teaching something new.

### Animation

Animation throughout the design is often a good choice. When animating always attempt to keep your frames per second above sixty or at least match the device's refresh rate.

Animations should guide the user but not block or delay interaction. Don't needlessly add flourishes and animate every element in a transition. Show the user what affect their action created and let them move on with using the app.

### Java Version

Target Java 8. Lambdas are your friend and just because Android is behind the times doesn't mean your code should suffer.

## Java Language Rules

Android follows standard Java coding conventions with the additional rules described below.

### Don't Ignore Exceptions

It can be tempting to write code that completely ignores an exception, such as:

```java
void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) { }
}
```

Do not do this. While you may think your code will never encounter this error condition or that it is not important to handle it, ignoring exceptions as above creates mines in your code for someone else to trigger some day. You must handle every Exception in your code in a principled way; the specific handling varies depending on the case.

Anytime somebody has an empty catch clause they should have a creepy feeling. There are definitely times when it is actually the correct thing to do, but at least you have to think about it. In Java you can't escape the creepy feeling. -James Gosling

Acceptable alternatives (in order of preference) are:

* Throw the exception up to the caller of your method.
```java
void setServerPort(String value) throws NumberFormatException {
    serverPort = Integer.parseInt(value);
}
```

* Throw a new exception that's appropriate to your level of abstraction.
```java
void setServerPort(String value) throws ConfigurationException {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) {
        throw new ConfigurationException("Port " + value + " is not valid.");
    }
}
```

* Handle the error gracefully and substitute an appropriate value in the catch {} block.
```java
/** Set port. If value is not a valid number, 80 is substituted. */

void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) {
        serverPort = 80;  // default port for server
    }
}
```

* Catch the Exception and throw a new RuntimeException. This is dangerous, so do it only if you are positive that if this error occurs the appropriate thing to do is crash.
```java
/** Set port. If value is not a valid number, die. */

void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) {
        throw new RuntimeException("port " + value " is invalid, ", e);
    }
}
```

> Note The original exception is passed to the constructor for RuntimeException. If your code must compile under Java 1.3, you must omit the exception that is the cause.

* As a last resort, if you are confident that ignoring the exception is appropriate then you may ignore it, but you must also comment why with a good reason:
```java
/** If value is not a valid number, original port number is used. */
void setServerPort(String value) {
    try {
        serverPort = Integer.parseInt(value);
    } catch (NumberFormatException e) {
        // Method is documented to just ignore invalid user input.
        // serverPort will just be unchanged.
    }
}
```

### Don't Catch Generic Exception

It can also be tempting to be lazy when catching exceptions and do something like this:

```java
try {
    someComplicatedIOFunction();        // may throw IOException
    someComplicatedParsingFunction();   // may throw ParsingException
    someComplicatedSecurityFunction();  // may throw SecurityException
    // phew, made it all the way
} catch (Exception e) {                 // I'll just catch all exceptions
    handleError();                      // with one generic handler!
}
```

Do not do this. In almost all cases it is inappropriate to catch generic Exception or Throwable (preferably not Throwable because it includes Error exceptions). It is very dangerous because it means that Exceptions you never expected (including RuntimeExceptions like ClassCastException) get caught in application-level error handling. It obscures the failure handling properties of your code, meaning if someone adds a new type of Exception in the code you're calling, the compiler won't help you realize you need to handle the error differently. In most cases you shouldn't be handling different types of exception the same way.

The rare exception to this rule is test code and top-level code where you want to catch all kinds of errors (to prevent them from showing up in a UI, or to keep a batch job running). In these cases you may catch generic Exception (or Throwable) and handle the error appropriately. Think very carefully before doing this, though, and put in comments explaining why it is safe in this place.

Alternatives to catching generic Exception:

* Catch each exception separately as separate catch blocks after a single try. This can be awkward but is still preferable to catching all Exceptions. Beware repeating too much code in the catch blocks.
* Refactor your code to have more fine-grained error handling, with multiple try blocks. Split up the IO from the parsing, handle errors separately in each case.
* Rethrow the exception. Many times you don't need to catch the exception at this level anyway, just let the method throw it.

Remember: exceptions are your friend! When the compiler complains you're not catching an exception, don't scowl. Smile: the compiler just made it easier for you to catch runtime problems in your code.

### Don't Use Finalizers

Finalizers are a way to have a chunk of code executed when an object is garbage collected. While they can be handy for doing cleanup (particularly of external resources), there are no guarantees as to when a finalizer will be called (or even that it will be called at all).

Android doesn't use finalizers. In most cases, you can do what you need from a finalizer with good exception handling. If you absolutely need it, define a close() method (or the like) and document exactly when that method needs to be called (see InputStream for an example). In this case it is appropriate but not required to print a short log message from the finalizer, as long as it is not expected to flood the logs.

### Fully Qualify Imports

When you want to use class Bar from package foo,there are two possible ways to import it:

* `import foo.*;`<br>
  Potentially reduces the number of import statements.
* `import foo.Bar;`<br>
  Makes it obvious what classes are actually used and the code is more readable for maintainers.

Use `import foo.Bar;` for importing all Android code. An explicit exception is made for java standard libraries (`java.util.*`, `java.io.*`, etc.) and unit test code (`junit.framework.*`).

## Java Library Rules

There are conventions for using Android's Java libraries and tools. In some cases, the convention has changed in important ways and older code might use a deprecated pattern or library. When working with such code, it's okay to continue the existing style. When creating new components however, never use deprecated libraries.

## Java Style Rules

### On Javadoc Standard Comments

I am of the opinion that documentation is generally a sign of poor naming, overly-complex methods, poor architecture, or other [code smell](https://en.wikipedia.org/wiki/Code_smell).

There are, of course, times when some documentation is necessary. Explaining a convoluted workaround, an inherited issue that will be later resolved, something that goes against existing style guides for good reason, and on. In these cases a single-line, sub-50 character note should do. If not, open a new issue to address the root cause and reference the issue in a comment.

### Write Short Methods

When feasible, keep methods small and focused. We recognize that long methods are sometimes appropriate, so no hard limit is placed on method length. If a method exceeds 40 lines or so, think about whether it can be broken up without harming the structure of the program.

### Define Fields in Standard Places

Define fields either at the top of the file or immediately before the methods that use them.

### Limit Variable Scope

Keep the scope of local variables to a minimum. By doing so, you increase the readability and maintainability of your code and reduce the likelihood of error. Each variable should be declared in the innermost block that encloses all uses of the variable.

Local variables should be declared at the point they are first used. Nearly every local variable declaration should contain an initializer. If you don't yet have enough information to initialize a variable sensibly, postpone the declaration until you do.

The exception is try-catch statements. If a variable is initialized with the return value of a method that throws a checked exception, it must be initialized inside a try block. If the value must be used outside of the try block, then it must be declared before the try block, where it cannot yet be sensibly initialized:

```java
// Instantiate class cl, which represents some sort of Set
Set s = null;
try {
    s = (Set) cl.newInstance();
} catch(IllegalAccessException e) {
    throw new IllegalArgumentException(cl + " not accessible");
} catch(InstantiationException e) {
    throw new IllegalArgumentException(cl + " not instantiable");
}

// Exercise the set
s.addAll(Arrays.asList(args));
```

However, even this case can be avoided by encapsulating the try-catch block in a method:

```java
Set createSet(Class cl) {
    // Instantiate class cl, which represents some sort of Set
    try {
        return (Set) cl.newInstance();
    } catch(IllegalAccessException e) {
        throw new IllegalArgumentException(cl + " not accessible");
    } catch(InstantiationException e) {
        throw new IllegalArgumentException(cl + " not instantiable");
    }
}

...

// Exercise the set
Set s = createSet(cl);
s.addAll(Arrays.asList(args));
```

Loop variables should be declared in the for statement itself unless there is a compelling reason to do otherwise:

```java
for (int i = 0; i < n; i++) {
    doSomething(i);
}
```

and

```java
for (Iterator i = c.iterator(); i.hasNext(); ) {
    doSomethingElse(i.next());
}
```

### Order Import Statements

The ordering of import statements is:

1. Android imports
1. Imports from third parties (`com`, `junit`, `net`, `org`)
1. `java` and `javax`
1. Same project imports

To exactly match the IDE settings, the imports should be:

* Alphabetical within each grouping, with capital letters before lower case letters (e.g. Z before a).
* Separated by a blank line between each major grouping (android, com, junit, net, org, java, javax).

Originally, there was no style requirement on the ordering, meaning IDEs were either always changing the ordering or IDE developers had to disable the automatic import management features and manually maintain the imports. This was deemed bad. When java-style was asked, the preferred styles varied wildly and it came down to Android needing to simply "pick an ordering and be consistent." So we chose a style, updated the style guide, and made the IDEs obey it. We expect that as IDE users work on the code, imports in all packages will match this pattern without extra engineering effort.

This style was chosen such that:

* The imports people want to look at first tend to be at the top (`android`).
* The imports people want to look at least tend to be at the bottom (`java`).
* Humans can easily follow the style.
* IDEs can follow the style.

Static imports should reside above all normal imports.

### Use Spaces for Indentation

We use four (4) space indents for blocks and never tabs. When in doubt, be consistent with the surrounding code.

We use eight (8) space indents for line wraps, including function calls and assignments. For example, this is correct:

```java
Instrument i =
        someLongExpression(that, wouldNotFit, on, one, line);
```

and this is not correct:

```java
Instrument i =
    someLongExpression(that, wouldNotFit, on, one, line);
```

### Field Naming Conventions

All field naming is [lower camel case](https://en.wikipedia.org/wiki/CamelCase#Variations_and_synonyms) with the exception of static final fields, which are [screaming snake case](https://en.wikipedia.org/wiki/Snake_case).

For example:

```java
public class MyClass {
    public static final int SOME_CONSTANT = 42;
    public int publicField;
    private static MyClass singleton;
    int packagePrivate;
    private int private;
    protected int protected;
}
```

### Class member ordering

There is no single correct solution for this but using a __logical__ and __consistent__ order will improve code learnability and readability. It is recommendable to use the following order:

1. Constants
2. Fields
3. Constructors
4. Override methods and callbacks (public or private)
5. Public methods
6. Private methods
7. Inner classes or interfaces

Example:

```java
public class MainActivity extends Activity {

  private String mTitle;
    private TextView mTextViewTitle;

    public void setTitle(String title) {
      mTitle = title;
    }

    @Override
    public void onCreate() {
        ...
    }

    private void setUpView() {
        ...
    }

    static class AnInnerClass {

    }

}
```

If your class is extending an __Android component__ such as an Activity or a Fragment, it is a good practice to order the override methods so that they __match the component's lifecycle__. For example, if you have an Activity that implements `onCreate()`, `onDestroy()`, `onPause()` and `onResume()`, then the correct order is:

```java
public class MainActivity extends Activity {

  //Order matches Activity lifecycle
    @Override
    public void onCreate() {}

    @Override
    public void onResume() {}

    @Override
    public void onPause() {}

    @Override
    public void onDestroy() {}

}
```

### Parameter ordering in methods

When programming for Android, it is quite common to define methods that take a `Context`. If you are writing a method like this, then the __Context__ must be the __first__ parameter.

The opposite case are __callback__ interfaces that should always be the __last__ parameter.

Examples:

```java
// Context always goes first
public User loadUser(Context context, int userId);

// Callbacks always go last
public void loadUserAsync(Context context, int userId, UserCallback callback);
```

### Use Standard Brace Style

Braces do not go on their own line; they go on the same line as the code before them:

```java
class MyClass {
    int func() {
        if (something) {
            // ...
        } else if (somethingElse) {
            // ...
        } else {
            // ...
        }
    }
}
```

We require braces around the statements for a conditional. Exception: If the entire conditional (the condition and the body) fit on one line, you may (but are not obligated to) put it all on one line. For example, this is acceptable:

```java
if (condition) {
    body();
}
```

and this is acceptable:

```java
if (condition) body();
```

but this is not acceptable:

```java
if (condition)
    body();  // bad!
```

### Arguments in Fragments and Activities

When data is passed into an `Activity `or `Fragment` via an `Intent` or a `Bundle`, the keys for the different values __must__ follow the rules described in the section above.

When an `Activity` or `Fragment` expects arguments, it should provide a `public static` method that facilitates the creation of the relevant `Intent` or `Fragment`.

In the case of Activities the method is usually called `getStartIntent()`:

```java
public static Intent getStartIntent(Context context, User user) {
  Intent intent = new Intent(context, ThisActivity.class);
  intent.putParcelableExtra(EXTRA_USER, user);
  return intent;
}
```

For Fragments it is named `newInstance()` and handles the creation of the Fragment with the right arguments:

```java
public static UserFragment newInstance(User user) {
  UserFragment fragment = new UserFragment;
  Bundle args = new Bundle();
  args.putParcelable(ARGUMENT_USER, user);
  fragment.setArguments(args)
  return fragment;
}
```

__Note 1__: These methods should go at the top of the class before `onCreate()`.

__Note 2__: If we provide the methods described above, the keys for extras and arguments should be `private` because there is not need for them to be exposed outside the class.

### Limit Line Length

Each line of text in your code should be at most 150 characters long. While much discussion has surrounded this rule, the decision remains that 150 characters is the maximum _with the following exceptions_:

* If a comment line contains an example command or a literal URL longer than 150 characters, that line may be longer than 150 characters for ease of cut and paste.
* Import lines can go over the limit because humans rarely see them (this also simplifies tool writing).

#### Line-wrapping strategies

There isn't an exact formula that explains how to line-wrap and quite often different solutions are valid. However there are a few rules that can be applied to common cases.

__Break at operators__

When the line is broken at an operator, the break comes __before__ the operator. For example:

```java
int longName = anotherVeryLongVariable + anEvenLongerOne - thisRidiculousLongOne
        + theFinalOne;
```

__Assignment Operator Exception__

An exception to the `break at operators` rule is the assignment operator `=`, where the line break should happen __after__ the operator.

```java
int longName =
        anotherVeryLongVariable + anEvenLongerOne - thisRidiculousLongOne + theFinalOne;
```

__Method chain case__

When multiple methods are chained in the same line - for example when using Builders - every call to a method should go in its own line, breaking the line before the `.`

```java
Picasso.with(context).load("http://ribot.co.uk/images/sexyjoe.jpg").into(imageView);
```

```java
Picasso.with(context)
        .load("http://ribot.co.uk/images/sexyjoe.jpg")
        .into(imageView);
```

__Long parameters case__

When a method has many parameters or its parameters are very long, we should break the line after every comma `,`

```java
loadPicture(context, "http://ribot.co.uk/images/sexyjoe.jpg", mImageViewProfilePicture, clickListener, "Title of the picture");
```

```java
loadPicture(context,
        "http://ribot.co.uk/images/sexyjoe.jpg",
        mImageViewProfilePicture,
        clickListener,
        "Title of the picture");
```

### RxJava chains styling

Rx chains of operators require line-wrapping. Every operator must go in a new line and the line should be broken before the `.`

```java
public Observable<Location> syncLocations() {
    return mDatabaseHelper.getAllLocations()
            .concatMap(new Func1<Location, Observable<? extends Location>>() {
                @Override
                 public Observable<? extends Location> call(Location location) {
                     return mRetrofitService.getLocation(location.id);
                 }
            })
            .retry(new Func2<Integer, Throwable, Boolean>() {
                 @Override
                 public Boolean call(Integer numRetries, Throwable throwable) {
                     return throwable instanceof RetrofitError;
                 }
            });
}
```

### Annotations

According to the Android code style guide, the standard practices for some of the predefined annotations in Java are:

* `@Override`: The @Override annotation __must be used__ whenever a method overrides the declaration or implementation from a super-class. For example, if you use the @inheritdocs Javadoc tag, and derive from a class (not an interface), you must also annotate that the method @Overrides the parent class's method.

* `@SuppressWarnings`: The @SuppressWarnings annotation should only be used under circumstances where it is impossible to eliminate a warning. If a warning passes this "impossible to eliminate" test, the @SuppressWarnings annotation must be used, so as to ensure that all warnings reflect actual problems in the code.

More information about annotation guidelines can be found [here](http://source.android.com/source/code-style.html#use-standard-java-annotations).

__Classes, Methods and Constructors__

When annotations are applied to a class, method, or constructor, they are listed after the documentation block and should appear as __one annotation per line__ .

```java
/* This is the documentation block about the class */
@AnnotationA
@AnnotationB
public class MyAnnotatedClass { }
```

__Fields__

Annotations applying to fields should be listed __on the same line__, unless the line reaches the maximum line length.

```java
@Nullable @Mock DataManager mDataManager;
```

### Treat Acronyms as Words

Treat acronyms and abbreviations as words in naming variables, methods, and classes to make names more readable:

| Good | Bad |
| ---- | --- |
| XmlHttpRequest | XMLHTTPRequest |
| getCustomerId | getCustomerID |
| class Html | class HTML |
| String url | String URL |
| long id | long ID |

As both the JDK and the Android code bases are very inconsistent around acronyms, it is virtually impossible to be consistent with the surrounding code. Therefore, always treat acronyms as words.

### Use TODO Comments

Use TODO comments for code that is temporary, a short-term solution, or good-enough but not perfect. TODOs should include the string TODO in all caps, followed by a colon:

```java
// TODO: Remove this code after the UrlTable2 has been checked in.
```
and

```java
// TODO: Change this to use a flag instead of a constant.
```

If your TODO is of the form "At a future date do something" make sure that you either include a very specific date ("Fix by November 2005") or a very specific event ("Remove this code after all production mixers understand protocol V7.").

### Log Sparingly

While logging is necessary, it has a significantly negative impact on performance and quickly loses its usefulness if not kept reasonably terse. The logging facilities provides five different levels of logging:

* `ERROR`: Use when something fatal has happened, i.e. something will have user-visible consequences and won't be recoverable without explicitly deleting some data, uninstalling applications, wiping the data partitions or reflashing the entire device (or worse). This level is always logged. Issues that justify some logging at the ERROR level are typically good candidates to be reported to a statistics-gathering server.
* `WARNING`: Use when something serious and unexpected happened, i.e. something that will have user-visible consequences but is likely to be recoverable without data loss by performing some explicit action, ranging from waiting or restarting an app all the way to re-downloading a new version of an application or rebooting the device. This level is always logged. Issues that justify some logging at the WARNING level might also be considered for reporting to a statistics-gathering server.
* `INFORMATIVE`: Use to note that something interesting to most people happened, i.e. when a situation is detected that is likely to have widespread impact, though isn't necessarily an error. Such a condition should only be logged by a module that reasonably believes that it is the most authoritative in that domain (to avoid duplicate logging by non-authoritative components). This level is always logged.
* `DEBUG`: Use to further note what is happening on the device that could be relevant to investigate and debug unexpected behaviors. You should log only what is needed to gather enough information about what is going on about your component. If your debug logs are dominating the log then you probably should be using verbose logging.<br>
This level will be logged, even on release builds, and is required to be surrounded by an `if (LOCAL_LOG)` or `if (LOCAL_LOGD)` block, where `LOCAL_LOG[D]` is defined in your class or subcomponent, so that there can exist a possibility to disable all such logging. There must therefore be no active logic in an `if (LOCAL_LOG)` block. All the string building for the log also needs to be placed inside the `if (LOCAL_LOG)` block. The logging call should not be re-factored out into a method call if it is going to cause the string building to take place outside of the `if (LOCAL_LOG)` block.<br>
There is some code that still says `if (localLOGV)`. This is considered acceptable as well, although the name is nonstandard.
* VERBOSE: Use for everything else. This level will only be logged on debug builds and should be surrounded by an `if (LOCAL_LOGV)` block (or equivalent) so it can be compiled out by default. Any string building will be stripped out of release builds and needs to appear inside the `if (LOCAL_LOGV)` block.

_Notes:_

* Within a given module, other than at the VERBOSE level, an error should only be reported once if possible. Within a single chain of function calls within a module, only the innermost function should return the error, and callers in the same module should only add some logging if that significantly helps to isolate the issue.
* In a chain of modules, other than at the VERBOSE level, when a lower-level module detects invalid data coming from a higher-level module, the lower-level module should only log this situation to the DEBUG log, and only if logging provides information that is not otherwise available to the caller. Specifically, there is no need to log situations where an exception is thrown (the exception should contain all the relevant information), or where the only information being logged is contained in an error code. This is especially important in the interaction between the framework and applications, and conditions caused by third-party applications that are properly handled by the framework should not trigger logging higher than the DEBUG level. The only situations that should trigger logging at the INFORMATIVE level or higher is when a module or application detects an error at its own level or coming from a lower level.
* When a condition that would normally justify some logging is likely to occur many times, it can be a good idea to implement some rate-limiting mechanism to prevent overflowing the logs with many duplicate copies of the same (or very similar) information.
* Losses of network connectivity are considered common, fully expected, and should not be logged gratuitously. A loss of network connectivity that has consequences within an app should be logged at the DEBUG or VERBOSE level (depending on whether the consequences are serious enough and unexpected enough to be logged in a release build).
* Having a full filesystem on a filesystem that is accessible to or on behalf of third-party applications should not be logged at a level higher than INFORMATIVE.
* Invalid data coming from any untrusted source (including any file on shared storage, or data coming through just about any network connection) is considered expected and should not trigger any logging at a level higher than DEBUG when it's detected to be invalid (and even then logging should be as limited as possible).
* Keep in mind that the + operator, when used on Strings, implicitly creates a StringBuilder with the default buffer size (16 characters) and potentially other temporary String objects, i.e. that explicitly creating StringBuilders isn't more expensive than relying on the default '+' operator (and can be a lot more efficient in fact). Keep in mind that code that calls Log.v() is compiled and executed on release builds, including building the strings, even if the logs aren't being read.
* Any logging that is meant to be read by other people and to be available in release builds should be terse without being cryptic, and should be reasonably understandable. This includes all logging up to the DEBUG level.
* When possible, logging should be kept on a single line if it makes sense. Line lengths up to 80 or 100 characters are perfectly acceptable, while lengths longer than about 130 or 160 characters (including the length of the tag) should be avoided if possible.
* Logging that reports successes should never be used at levels higher than VERBOSE.
* Temporary logging used to diagnose an issue that is hard to reproduce should be kept at the DEBUG or VERBOSE level and should be enclosed by if blocks that allow for disabling it entirely at compile time.
B* e careful about security leaks through the log. Private information should be avoided. Information about protected content must definitely be avoided. This is especially important when writing framework code as it's not easy to know in advance what will and will not be private information or protected content.
* `System.out.println()` (or `printf()` for native code) should never be used. System.out and System.err get redirected to /dev/null, so your print statements will have no visible effects. However, all the string building that happens for these calls still gets executed.
* _The golden rule of logging is that your logs may not unnecessarily push other logs out of the buffer, just as others may not push out yours._

### Be Consistent

Our parting thought: BE CONSISTENT. If you're editing code, take a few minutes to look at the surrounding code and determine its style. If that code uses spaces around the if clauses, you should too. If the code comments have little boxes of stars around them, make your comments have little boxes of stars around them too.

The point of having style guidelines is to have a common vocabulary of coding, so people can concentrate on what you're saying, rather than on how you're saying it. We present global style rules here so people know the vocabulary, but local style is also important. If the code you add to a file looks drastically different from the existing code around it, it throws readers out of their rhythm when they go to read it. Try to avoid this.

## XML style rules

As noted in the opening text about user interfaces you should prefer XML layouts.

### Use self closing tags

When an XML element doesn't have any contents, you __must__ use self closing tags.

This is good:

```xml
<TextView
  android:id="@+id/text_view_profile"
  android:layout_width="wrap_content"
  android:layout_height="wrap_content" />
```

This is __bad__ :

```xml
<!-- Don\'t do this! -->
<TextView
    android:id="@+id/text_view_profile"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" >
</TextView>
```


### Resources naming

Resource IDs and names are written in __lowercase_underscore__.

## Javatests Style Rules

Follow test method naming conventions and use an underscore to separate what is being tested from the specific case being tested. This style makes it easier to see exactly what cases are being tested. For example:

```java
testMethod_specificCase1 testMethod_specificCase2

void testIsDistinguishable_protanopia() {
    ColorMatcher colorMatcher = new ColorMatcher(PROTANOPIA)
    assertFalse(colorMatcher.isDistinguishable(Color.RED, Color.BLACK))
    assertTrue(colorMatcher.isDistinguishable(Color.X, Color.Y))
}
```

### What not to test

Don't test basic, Plain Old Java Objects (POGOs). If you can't trust the system to successfully use getters and setters there are bigger issues.

Test everything else.

Runners vary for what is to be tested. Don't test the UI with JUnit. Don't test objects with a UI test runner.
