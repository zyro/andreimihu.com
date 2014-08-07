+++
date = "2013-10-17"
title = "Android: Always embed tabs in ActionBar"
description = "How to force embedding of navigation tabs in the Android ActionBar"
keywords = ["android", "tabs", "embed", "actionbar", "portrait", "orientation"]
categories = ["android"]
+++
When I chose to use a tab layout alongside the standard Android ActionBar for the home screen of my [CrunchBase app](https://github.com/zyro/crunchbased) I found it surprisingly difficult to get the behaviour I wanted: I expected tabs to be embedded in the ActionBar, subject to available space. Not so!

By default, Android will happily embed tabs into the ActionBar in landscape orientation, if it finds enough room. But in portrait mode they are displayed in their own bar and the large blank space left just above them is ignored.

I was keen to minimise the screen area used by navigation items, so I set out to find an answer...

{{% figure src="/images/2013-10-17-android-always-embed-tabs-in-actionbar-portrait1.jpg" caption="Default portrait orientation behaviour" %}}
{{% figure src="/images/2013-10-17-android-always-embed-tabs-in-actionbar-landscape1.jpg" caption="Default landscape orientation behaviour" %}}

---

### Not so simple

Unfortunately, my answer comes with a disclaimer of sorts.

This seems like a rather basic use case and I personally find it surprising the effect is so difficult to achieve. It is clearly not a limitation of the UI elements themselves, since there are no issues in landscape orientation. Whatever the case, this behaviour is clearly by design.

This means that short of replacing the ActionBar implementation with a more permissive solution, there appears to be no *guaranteed* solution.

---

### The code

This has been tested on Android 4.0.3 and above - API level 15+. Here's what it looks like:

``` java
public class AwesomeTabbedActivity extends Activity {

    @Override
    public void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.awesome_tabbed_layout);
        
        // Standard tabbed navigation setup.
        final ActionBar actionBar = getActionBar();
        actionBar.setDisplayShowTitleEnabled(true);
        actionBar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);
        
        // ...
        // Create tabs, fragments, pager and anything else needed.
        // ...
        
        forceTabs(); // Force tabs when activity starts.
    }
    
    @Override
    public void onConfigurationChanged(final Configuration config) {
        super.onConfigurationChanged(config);
        forceTabs(); // Handle orientation changes.
    }

    // This is where the magic happens!
    public void forceTabs() {
        try {
            final ActionBar actionBar = getActionBar();
            final Method setHasEmbeddedTabsMethod = actionBar.getClass()
                .getDeclaredMethod("setHasEmbeddedTabs", boolean.class);
            setHasEmbeddedTabsMethod.setAccessible(true);
            setHasEmbeddedTabsMethod.invoke(actionBar, true);
        }
        catch(final Exception e) {
            // Handle issues as needed: log, warn user, fallback etc
            // This error is safe to ignore, standard tabs will appear.
        }
    }

}
```

Success! The tabs are now embedded in the ActionBar in portrait mode, while landscape orientation still works as expected. If there is insufficient space, the tabs will stack into a neat drop-down menu.

{{% figure src="/images/2013-10-17-android-always-embed-tabs-in-actionbar-portrait2.jpg" caption="Portrait with enough room" %}}
{{% figure src="/images/2013-10-17-android-always-embed-tabs-in-actionbar-portrait3.jpg" caption="Portrait with drop-down tabs" %}}

---

### Limitations

As with any circumvention of intended behaviour via reflection, this is not guaranteed to work on every combination of device and software, especially given Android's fragmented ecosystem.

Testing yielded promising results, with only one minor issue. On some devices the active tab indicator, or the active item in the tab drop-down if the tab list has been collapsed, may not be updated correctly if you have a `ViewPager` configured to allow you to swipe between tabs.

There is also a chance that in some future API version the `setHasEmbeddedTabs` method being invoked reflectively will be renamed, moved or removed entirely, rendering the code snippet above ineffective.

---

### Conclusion

Despite these drawbacks, in this scenario reflection is still the most reliable option that does not involve bringing in external libraries.

This issue definitely proved an interesting insight into Android UI code and its inner workings, even if I eventually decided against applying this solution to the [released version of my app](https://play.google.com/store/apps/details?id=com.github.zyro.crunchbased).