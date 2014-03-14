---
layout: post
title: "Hiding the Windows arrow cursor"
description: ""
tags:
  - windows
  - winapi
  - visualc++
---
{% include JB/setup %}

For a project I needed to be able to hide the cursor globally on Windows. The easiest
and less intrusive way to this was by simply replacing the arrow cursor by a transparent
1px cursor. This obviously doesn't disable the cursor, but it gets the job done when
all you care about is to visually hide it in all applications.
The example below only hides the [IDC_ARROW](http://msdn.microsoft.com/en-us/library/windows/desktop/ms648391.aspx)
cursor, you would have to replace multiple cursors if you never want to see any mouse pointer.

To accomplish this you first need to get the handle of the system cursor you want to hide, IDC_ARROW in this case.
Then you will need to make a copy using [CopyCursor](http://msdn.microsoft.com/en-us/library/windows/desktop/ms648384.aspx)
so you can restore it later on.

{% highlight c %}
// Save a copy of the default cursor
HANDLE arrowHandle = LoadImage(NULL, MAKEINTRESOURCE(IDC_ARROW), IMAGE_CURSOR, 0, 0, LR_SHARED);
HCURSOR hcArrow = CopyCursor(arrowHandle);

// Set the cursor to a transparent one to emulate no cursor
// IDC_NOCURSOR is defined in resources and is a transparent .cur file
HANDLE noCursorHandle = LoadImage(GetModuleHandle(NULL), MAKEINTRESOURCE(IDC_NOCURSOR), IMAGE_CURSOR, 0, 0, NULL);
HCURSOR noCursor = CopyCursor(noCursorHandle);
SetSystemCursor(noCursor, OCR_NORMAL);
{% endhighlight %}

When you no longer need to hide the cursor, you simply set the copy of the original cursor
again as system cursor and you destroy the transparent cursor.

{% highlight c %}
SetSystemCursor(hcArrow, OCR_NORMAL);
DestroyCursor(hcArrow);
{% endhighlight %}

References:

1. [MSDN Cursor Reference](http://msdn.microsoft.com/en-us/library/windows/desktop/ff468814.aspx)

2. [The transparent cursor file I used](/assets/files/nocursor.cur)
