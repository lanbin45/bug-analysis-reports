**UAT BUG \#27759 ANALYSIS REPORT**

Background 
===========

The application Utilization Analysis Tool (UAT, Hereinafter referred to as UAT)
home page and spotlight page has become blank screen (show in Fig.1) while hit
refresh button or change analysis time interval.

![](media/2d83e61b0bd1c4312680c69dfa227117.png)

>   Fig.1. Bug\#27759 test result

For the case in Fig.1, the client side has corrupted, we’ve checked the node.exe
process in task manager and it runs well. So we prefer to consider it as a
client-side memory leak.

Investigation
=============

Test
----

Considering the resource life cycle and any other contingency factors affect
single test result, we write a scripts for repetitive auto test using Sikulix to
simulate mouse click and keyboard input.

### Test environment 

>   DELL T7810

>   Intel(R) Xeon(R) E5-2609 v3 \@ 1.90GHz 8Cores

>   RAM 32.0GB

>   Win10 64bit

>   Vitrea 7.4.0.87

TestData: RIGS\_anonymizeddata\_V7.3FT+\_20160913\_with\_XA\_real.bak

DataSize: 231,904 kB

### Test steps 

According our former auto tests result, we select a test scene that always cause
dump to validate our solution:

>   1. Import test data to VSP;

>   2. Launch UAT from Appshell;

>   3. Select analysis conditions:

>   Analysis starts: 2016-04-12, Analysis ends: 2017-10-29, regions: all,
>   patient type: all, x-axis: protocol;

>   4. Click refresh button and reload data every 30s;

 Home Page Refresh
------------------

### Problem Location

![](media/a2e5fc8c7519bf874b7b1244da47f690.png)

![](media/85cac37f8a7672eb77a0e2e41be90588.png)

Generally, we use timeline tab in chrome developer tools to record the JS Heap,
Documents, Nodes and Listeners in a refresh operation. To insure that Garbage
Collection (GC) in chrome has taken effect, we click in timeline tab before each
operation to force GC work. See result in Fig2:

Fig2. Timeline after a refresh operation

The timeline records shows distinctly that:

1.  JS heap increase from 18.5MB to 47.8MB, some components may not recycle
    correctly and cause a huge memory leak when hitting refresh button;

2.  Nodes increase from 4645 to 6298, document elements may not release.

Usually, Ext.js use Ext.ComponentManager and Ext.ClassManager classes to manager
components and data (model and store). Firstly, we use Ext.ComponentManager.all
to get all components before and after refresh, and we’ve find components’ count
increase 5 (while we expected 0) after every operation. See in Fig.3. Then we
check Ext.ComponentManager.map to verify which component created.

>   [./media/image4.png](./media/image4.png)

>   Fig3. Components management before and after refresh operation

Unfortunately, we’ve compared hundreds of components, and find 5 labels created.
Considering the JS heap increment (about 30Mb), the size of 5 labels is not a
crucial factor that lead to memory leak. This was verified after we recycled the
increased components, memory leak decrease little (even make no difference).

Then we used Profile tool in chrome developer tools and take heap snapshots
before and after refresh operation to see which constructor is not deleted
correctly. See in Fig.4, select Snapshot 4 and choose Comparison mode, the tool
page shows constructors’ counts and size by create, deleted, delta.

Normally, if a constructor’s \#deleted column is 0, it will be considered as a
memory leaked constructor. Notice that some SVG (Scalable Vector Graphics, SVG)
related constructors created but never deleted! The SVG elements did not
recycle, this indicated that “old” charts may not destroyed while new charts
created when refresh charts in home page.

![](media/22dc055521b589f8b5cdb1d226da6087.png)

Fig.4. Heap Snapshot comparison

In home page, there’s 3 charts: Dosage, Duration and Utilization. We take Dosage
boxplot chart to illustrate currently refresh flow:

1.  Get the panel that contains dosage charts;

2.  Remove all child items from the panel;

3.  Add panels and display charts.

Code segment shows in Fig.5.

![](media/e28d3ef9dd0dbec10e18c77dc6ab87ce.png)

The logic here seems ok. “old” charts destroyed in panel.removeAll(); look up
Sencha document:

![](media/989fb48b7f0103846aa435d8a0f9b54e.png)

Actually, the panel.removeAll() remove all components from panel, and the
components destroyed, but Highcharts still get the SVG elements’ reference, the
SVG elements still remain in memory, GC cannot recycle them.

### Solutions

After locating the problem, we added a SVG elements recycle process use
Highcharts.destroy() before remove all components from panel, and explicitly
destroy child component while remove them.

![](media/1e95f2f0dea39f309f52ed832eb8a51b.png)

Still, we run timeline to record JS heap of refresh operation, results shows in
Fig.5:

![](media/00804ae9f74c40c197695fdf0bdef79e.png)

Fig.5 Timeline result after modified

To validate the client side modified take effect, we repeat test steps in 2.1.2
and refresh every 30 seconds, and use task manager to observe memory usage of
appshell. After about 15 hours test (from 2017.03.23 19:14 to 2017.04.24 10:06),
the memory usage of appshell increase from 40,192k to 42132k, and client side
work well.

Spotlight Page Refresh
----------------------

### Problem Location

![](media/a85b0a3be08d866baedf35b642eee3cd.png)

We also notice that spotlight page occurred same result after hitting refresh
many time, we keep same analysis conditions and test in spotlight.

Fig.6 Spotlight page

The spotlight page (shows in Fig.6) include 2 parts: chart and grid data view.
The chart part use the same logic as home page, and has been solved in chapter
2.2. After solved home page refresh issue, we comment code of chart refresh and
abandon refresh chart logic, thus, when hitting refresh button in spotlight
page, only grid view part remains and we repeat hit refresh page, after 100
times refresh, the page refresh slowly and crashed (turn blank screen) later.

Run timeline analysis and take heap snapshot in chrome developer tools, result
shows in Fig.7 and Fig.8:

>   [./media/image11.png](./media/image11.png)

>   Fig.7 Spotlight page timeline analysis

>   [./media/image12.png](./media/image12.png)

>   Fig.8 Spotlight page heap snap shot

Fig.7 shows JS heap increase slowly after every refresh operation. In Fig.8,
we’ve noticed that the second constructor is store data. Click and check detail
(shows in Fig.9)

>   [./media/image13.png](./media/image13.png)

>   Fig.9 Spotlight page heap snap shot detail

![](media/090ed9c8031ee3612e4537e8104a12bc.png)

Generally, the yellow background with red font constructor (like ) may greatly
doubt to be memory leak object. And the store data point to grid data view
store.

>   The original logic for grid data view is using an anonymous
>   Ext.data.Arraystore to store outlier data:

>   [./media/image15.png](./media/image15.png)

This is very suspicious, it looks like a Model definition like the one we have
specified but why so many of them? The solution is found in how the Store deals
with anonymous models (using the ‘fields’ config). It defines a new Model class
each time the store is created. A quick peek at Ext.ClassManager.classes
verifies this:

![](media/b44c106516cda1f05ee06ade037c4f95.png)

It means while update grid data view, the anonymous model (outlierStore here)
create an Implicit Model every time and did not destroy them. The outlierStore
data remains in memory leads to a memory leak.

### Solution 

How to solve this problem?

Here we use a store.storeId to register the created store and add store
configuration {autoDestroy: true}, then the Ext.storeManagement will manger the
register store lifecycle.

![](media/d13bf2e1347da3742ba2ae44c7c7c6f8.png)

we run timeline to record JS heap of refresh operation, results shows in Fig.10:

![](media/5cc6f6682937002d28429eedf26766a4.png)

Fig.10 Timeline result after modified

Similarly, to validate the client side modified take effect, we repeat test
steps in 2.1.2 and refresh every 30 seconds, and use task manager to observe
memory usage of appshell. After about 12 hours test, the memory usage of
appshell increase from 38,433k to 43561k, and client side still work well.

Conclusion
==========

Tips
----

1.  **Release all reference before removing javascript Object;**

>   For this case, when remove SVG elements, even we use
>   Ext.panel.removeAll(true) to remove and destroy components on the panel, the
>   Extjs components destroyed, but Highcharts still get the reference of SVG,
>   thus SVG elements cannot recycle correctly. They remain in memory and cause
>   memory leak.

1.  **Carefully use anonymous store/model;**

>   When define a Ext.data.store class, use a storeId to register it in
>   Ext.storeManager, avoid to use anonymous store/model, because an anonymous
>   store/model will create an Implicit Model every time, which needs to be
>   destroyed manually.

How to locate memory leaks in Ext.js applications
-------------------------------------------------

A memory leak appears when a resource that’s no longer used by any part of your
application still occupies memory which will never be released. Determining what
is a memory leak isn’t always easy since we don’t have control over the browser
garbage collecting process. This prevents us from reliably knowing when the
browser is done cleaning up unused JS objects, DOM nodes etc. This means we have
to be extra careful and create solid test cases that can prove a memory
increasing pattern over time.

There are 2 useful tools use to locate memory leaks:

1.  **Ext.js classes**

>   Here are some specific classes we can use to check Ext.js application memory
>   leak:

>   **Ext.StoreManager** contains all the stores you’ve created in your
>   application, make sure it contains only stores you expect to see and destroy
>   any internal stores when they’re no longer used.

>   **Ext.ClassManager** contains all the classes defined in your application as
>   well as all the Sencha classes, make sure it contains only classes you
>   expect to see.

>   **Ext.Element.cache** contains all cached Ext.Element references. Avoid
>   using Element.down and similar APIs unless you really need an Ext.Element
>   instance.

>   **Ext.ComponentManager** contains all cached Component instances. If you
>   create a component and forget to destroy it, it’ll still be kept in the
>   cache.

>   **Ext.event.Dispatcher.getInstance()** If you forget to destroy an
>   Observable you’ll see an entry leaked in this object.

1.  **Chrome devTools**

>   **Timeline panel:** This centralized overview of your app’s activity helps
>   you analyze where time is spent on loading, scripting, rendering, and
>   painting. It has become a powerful tool to get the full story of your app’s
>   activity.

>   **Heap allocation profile** displays the location where the object is being
>   created and determines the retention path. In the following snapshot, the
>   top bar indicates the time at which the new object is found in the heap.
>   Take 2 snapshots before and after an operation then choose Comparison mode
>   to compare snapshots helps you find which constructor did not release
>   correctly.
