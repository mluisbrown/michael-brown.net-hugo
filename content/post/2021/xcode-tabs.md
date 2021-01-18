---
title: "Xcode 12 tabs"
date: 2021-01-17
tags:
- swift
- dev
---

Up until Xcode 12 was released, Xcode tabs were a pretty terrible user experience, especially if you also regularly used an actually good code editor like Sublime Text or Visual Studio Code and knew what a good tabs UX was like. In Xcode tabs you could open the same file in 2 (or more) different tabs, for instance, which is close to useless. Xcode tabs worked pretty much like document tabs in any other application. Unfortunately, this style of tabs is not great for editing code where you are constantly switching between many different files.

Xcode 12 introduced a new type of tabs called which I call "Editor Tabs". I'm not sure if they have an official name. Unfortunately, this change wasn't very well documented and the default settings made it easy to end up with the new style "editor" tabs nested in the old style "window" tabs, which is super confusing:

![Xcode tabs within tabs](/images/xcode-tabs/tabs-in-tabs.png)

If you happen to be one of those people who *like* the pre Xcode 12 tabs and never want to see the new style tabs, then check out [Jesse Squires' post](https://www.jessesquires.com/blog/2020/07/24/how-to-fix-the-incomprehensible-tabs-in-xcode-12/) about how to configure that.

However, if you want Xcode tabs to behave the same as any other decent code editor, I will show you how to configure that below.

Open Xcode Navigation Preferences, and set the following options:

* Navigation Style: Open in Tabs (open new files in a separate editor tab)
* Navigation: Uses Focused Editor (open in a tab the editor with the current focus)
* Optional Navigation: Uses Next Editor (open in the "next" editor if you hold down Option while opening the file)
* Double-click Navigation: Uses tab (opens a new tab - see below for explanation).

![Xcode settings](/images/xcode-tabs/xcode-settings.png)

Below the settings pickers, is a small description of what each file open action will do, and you will see that for "Click" it says "Shows preview in focused editor". If you have used Sublime Text, VS Code and others good code editors, you will know what "preview" means. If you don't already have a tab open for this file, it will open a new tab for it, but the title of the tab will be in *italics*. This is how to tell a preview tab from a "real" tab. If you don't make any edits to the file, and you open another file, the preview tab will be re-used, the previous file will be replaced by the new one. This preview mechanism exists to avoid opening hundreds of new tabs if you're just browsing through a bunch of source files without editing them, which would be really tedious.

You can convert a preview into a real tab either by making an edit in the file or by double-clicking its title. The last option in the settings above "Double-click Navigation: Uses tab" will also make any file you double-click on become an actual tab straight away, without first being a preview.

## Switching tabs

You can navigate back and forth between tabs using the Cmd+{ and Cmd+} shortcuts. The list of tabs is also scrollable with trackpad swipe gestures. Also, if you open a file which is already open, you will just be sent directly to the already open tab for it.

## Xcode Editors

Sometimes it's useful to have two editors side by side, so you can edit one file whilst referring to another. Xcode supports this using Editors. You can create a new Editor using the Ctrl+Cmd+T shortcut. Each Editor has it's own set of tabs. When you open a new file and you have the "Navigation" option above set to "Uses Focused Editor", the new file will open in a tab in the editor that has the current focus. Option opening a file (holding down Option whilst you open it) will, with the settings above, open the file in a new Editor if there is only one editor, but otherwise just use the 2nd editor. You can configure this to always open a new editor instead, although be aware that depending on the size of the Xcode window, there is a maximum of 3 or 4 editors.

## Old style tabs

Of course the old style Xcode tabs still exist. The Cmd+T shortcut will open a new one. But I avoid using them. They now have absolutely no use for me at all. In fact, it may be a good idea to delete the Cmd+T shortcut mapping altogether to avoid accidentally creating any old style tabs.