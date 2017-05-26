**PDM BUG \#26373 ANALYSIS REPORT**

Background 
===========

During V7.4 16H test on Oct 18th, reported that Patient Dose Management could
not refresh RDSR Info dialog correctly. However, in our side we could not
duplicate the defect.

And there is another UI display defect from FT site, which we could not
duplicate in our side also.

Investigation
=============

Test
----

At first, we simulated all types of user operations and did long time test by
auto test script (Sikulix framework to simulate mouse click and keyboard input).
And we found that:

-   Server side worked normally.

-   Client side error occurred randomly.

-   These errors are different from each time.

-   Most of the errors are Ext.js api call error, especially when entering
    detail window and switch analysis condition between “All Patient” and “This
    Patient”.

So, we doubt that there may be memory leak in code of detail window in PDM app.

Then, we simplified the auto test script, and create another scripts using wmi
package in python to record the resource (Peak Virtual Size, Peak Working Set
Size, Virtual Size, Working Set Size) usage of appshell.

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

>   2. Launch PDM from Appshell;

>   3. Click column where Patient ID = “F508293F” in patient list;

>   4. Click column where Operator = “O’Neal” in history list and enter into
>   detail window;

>   5. Click “All Patient” to switch tab;

>   6. Click “This Patient” to switch tab;

>   7. Click “Close” button;

>   8. Take a screenshot to see whether app dumped;

>   9. Wait 1min and loop from Step2-Step8;

 Problem location
-----------------

![](media/08b9248c2cbdce36d84e44a0f7feb805.png)

![](media/a2e5fc8c7519bf874b7b1244da47f690.png)

We used timeline tab in chrome developer tools to record the JS Heap, Documents,
Nodes and Listeners from enter detail window to close detail window. To insure
that Garbage Collection (GC) in chrome has taken effect, we click in timeline
tab before each operation to force GC work. See result in Fig1:

Fig1. Timeline from enter detail window to close detail window

We can see that after close detail window, JS heap increase from 12,589,824
bytes to 17,587,632 bytes, some nodes did not destroy after close. Thus we
ensure the memory leak exists in detail window.

In detail window, When export thumbnail of charts, the logic here:

Fig2. Export thumbnail image logic

The PDM client side used Ext.js as frame structure, when this bug occurs, we
firstly check the components’ count before enter and close detail window. In
Ext.js:

>   Ext.ComponentManager.getCount();

We add it before create detail window (function openWindow() in
../controller/grid/HistoryList.js) and after close detail window (function
onClose() in ../controller/detail/xa/window.js ), Thus we get the components’
count before enter and after close detail window (for 1 operation):

|    | Before enter | After close |
|----|--------------|-------------|
| XA | 175          | 269         |
| CT | 177          | 255         |

Then we use

>   Ext.ComponentManager.all;

to get all existed components. We’ve found all the components that are not
closed correctly, and we destroy them correctly in function onClose()
(../controller/detail/xa/window.js):

![](media/c8612fbd329c0524d6edab1c32e5fe77.png)

After doing this, JS heap has decrease from 17,587,632 to 16,227,348 bytes. It’s
effective, but **NOT ENOUGH**!

**Validation:**

We’ve run original and modified version 1 code to test whether our modification
take effect. Record test times, the resource usage before start auto test and
after app dumped. Test result shows here (size unit: bytes):

| Code version       | Run times | Peak Virtual Size   | Peak Working Set Size | Virtual Size        | Working Set Size    |
|--------------------|-----------|---------------------|-----------------------|---------------------|---------------------|
| Original           | 112       | 391204864 518893568 | 119624 241200         | 355028992 469618688 | 87801856 392278016, |
| modified version 1 | 124       | 391213233 529461248 | 119624 248888         | 364470272 603754496 | 91975680 377876480  |

Test result shows the modification of Chapter 2.1 takes work little, app still
exists memory leak.

Solutions
---------

We look back export thumbnail image logic, and notice that in Step 4 the canvas
did not destroy, and in debug mode we can query the canvases after close detail
window. We destroy them after finish export thumbnail:

>   \$('canvas').remove();

Still effective, but not enough! Memory leak still exist!

I’ve noticed that in resource tab in chrome developer tool, the images did not
destroy after close! And the images increase after each operation when enter
detail window. The image resources store as dataUrl (data:image/jpeg;base64,….).
The size of each image is about 40-60kB. See in Fig3.a and Fig3.b:

![](media/324df88b58af22c3ff0daf0fb7fbfa88.jpg)

![](media/cd7e1b24cf216de3fe2cf3c7fa759b90.jpg)

Fig3.a Resource tab after first time Fig3.b Resource tab after second time

enter and close detail window enter and close detail window

How about destroy the images that store in the chrome? Sound great! But
unfortunately, chrome may have some bugs here:

1.  Setting img.src to Data URI causes memory leak:

<https://bugs.chromium.org/p/chromium/issues/detail?id=309543&thanks=309543&ts=1382344039>

1.  Memory usage grows infinitely when changing img.src:

>   <https://bugs.chromium.org/p/chromium/issues/detail?id=114570>

1.  manipulating img.src via javascript will generate a massive memory leak:

>   <https://bugs.chromium.org/p/chromium/issues/detail?id=114570>

It seems that current vision chrome can not recycle dataUrl data correctly after
we explicit destroy it in javascript files.

### Solution 1 

In this case, we can not use dataUrl. How about using Blob object to store image
data? The Blob object could be revoked by chrome correctly! I’ve checked the
HTMLCanvasElement APIs, luckily, the HTMLCanvasElement.toBlob() api meet our
requirements.

In function exportThumbnail() (../controller/detail/xa/BaseChart.js), it is
modified as follow:

![](media/9619b148938bc0c0bbe7f80b9c4681d9.png)

In this case, after the images load in the thumbnail container panel, the blob
object is revoked use

>   URL.revokeObjectURL(url);

Take a timeline record, Images resource destroyed and memory leak stopped!

![](media/b6308727461fea0aa3266c45f1dd060a.png)

A way to see blob objects in chrome is type in “chrome://blob-internals/”. When
debug, we see the blob objects created:

![](media/6fef266318e08d9b478d9858574b236c.png)

And destroyed:

![](media/39631ff20a1ff51c320902b2de423efb.png)

Up to now, this problem seems to be solved! Client side runs perfectly in local
chrome environment!

**Validation:**

How about VSP environment? Unfortunately, Chrome supports
**HTMLCanvasElement.toBlob()after chrome/50.0.x version, vsp 7.3.0 appshell’s
version is chrome/47.0.x, did not support this feature.**

![](media/f85ab7f44c3e13b48b5bd6de52f66c16.png)

This solution may be used in VSP in further version, but did not work at current
VSP environment.

### Solution 2

Think differently, do we really need to convert svg html to image? How about
create a canvas in thumbnail container panel and draw svg on the canvas
directly? Have a try.

Create a canvas in thumbnail container panel:

![](media/dde7ec52da658904b36f1f0d2947f998.png)

The first problem we’ve meet is that original svg’s size didn’t fit the
thumbnail container panel. The svg viewBox attribute allows you to specify that
a given set of graphics stretch to fit a particular container element. The
canvg.js plugin also support scale svg size.

![](media/a24f7e2b41907a452ebaa86f8ee54742.png)

The thumbnail works correctly after resize the canvas. Take a timeline record to
check JS heap, Documents, Nodes and Listeners.

![](media/235b500a383e46feacf958a239bbf1eb.png)

The JS Heap increase from 12,665,118 bytes to 17,587,388 bytes. It seems not
work at all! And we’ve noticed that the Documents increase from 1 to 5. It goes
even worse!

I’ve doubt that the canvas did not destroy, query the canvas elements in parent
window (home page), returns null. We ensure the canvases destroyed after close
detail window, but it did not collect by Chrome GC!

Check the timeline records, I’ve noticed that after close detail window, the
application still call functions in canvg.js:

![](media/ad727e38f389314be110050c2ca0526f.png)

The canvg.js plugin seems to have some bug internal. I’ve checked the issues
list in github:

<https://github.com/canvg/canvg/issues>

None of these issues meet the memory leak problem. At last I have to check the
source code. The closure issues has been solved in renderCallback() in
exportThumbnail() :

![](media/e83cfa22a10ac32eca1077c9995580f0.png)

Problem still exist. Debug source code in steps, I’ve found that there is a
setInterval() function executed but did not clear interval. See from red box
below, this function called every 100/svg.FRAMERATE ms.

![](media/8b29b084df41c7783e653e32a273073e.png)

Thus the svg elements in yellow box did not release actually, the timer still
call the resources, Chrome GC would not destroy them either!

I’ve modified the canvg.js file, write a svg.stop() function to clear intervals,
and call svg.stop() after finish drawing.

![](media/9e4b49a7bac7263e638860d373910918.png)

Check the timeline record, the JS Heap, Documents, Nodes, Listeners get right
after close detail window. JS heap increase from 13,891,200 bytes to 14,146,208
bytes. Considering some resouces life cycle, the Chrome GC did not destroy them
immediately. It’s acceptable.

![](media/2e9cab99a4281586894def945218b925.png)

### Validation:

To validate our assumptions, we begin an auto test as before. App did not dumped
again ever, and Peak Virtual Size, Peak Working Set Size, Virtual Size, Working
Set Size steady on 410,000,000 bytes, 120,000 bytes, 380,000,000 bytes,
100,000,000 bytes each.

This version of the code runs perfectly in VSP, and solved the memory leak when
export thumbnail.

Conclusion
==========

We investigation the reason of client side memory leak related to detail window,
and provide 2 solutions to solve this problem. For the VSP appshell version
reason, we choose solution 2:

1.  **Change the export thumbnail logic to avoid chrome bugs related dataUrls:
    create a canvas in thumbnail container panel and draw svg on the canvas
    directly;**

2.  **Modified the third party plugin canvg.js file, write a svg.stop() function
    to clear intervals, and call svg.stop() after finish drawing to make sure
    the resources are completely released!**

Solution 2 has solved bug \#26737.

Other issues
============

Because we’ve modified the third party plugin canvg.js, some issues related with
canvg.js may meet memory leak both in PDM and UAT. Currently, we’ve found that
in PDM PDF export and UAT PDF export issues used the same logic to convert svg
to image, and we are set about to solve it!
