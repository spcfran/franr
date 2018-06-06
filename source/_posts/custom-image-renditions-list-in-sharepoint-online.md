---
title: Custom image renditions list in SharePoint Online
permalink: custom-image-renditions-list-in-sharepoint-online
date: 2016-03-21
tags:
---

Image Renditions in SharePoint are a really useful feature for both content editors and development of solutions, which can confuse the former when the list grows too big. In this article we'll propose a solution to gain control over this list and make it more user friendly by exploring how the ribbon works.

<!-- less --> 

Image renditions are a mechanism in SharePoint Online (and also 2013 on premises) that enables having different versions, of different sizes and shapes, of the same source image. I won't explain them in detail (check out [Waldek Mastykarz's article][1] if you are not familiar with them) but in short:

- The scaled down images are automatically generated and stored as actual files in a BLOB cache in the server, and the smaller the picture, the lower the payload through the wire. The gain in performance of serving an actual smaller image compared to the old dirty trick of sending the huge picture and scale down in CSS can be very significant.
- The different images can then be retrieved by requesting the URL of the original image with some querystring parameters.
- There's a list (an XML file, actually) defining the details of each rendition which is managed at site collection level by an admin, but content authors can specify the size and crop details (i.e. the "viewport") of each rendition for any image, by stretching and dragging around a rectangle in the rendition editor:

![Rendition editor](./rend-editarendition.jpg)

## Renditions, renditions everywhere

Because renditions are so handy, sometimes we can end up with a long list of them visible to the users, when in reality most of them are intended for internal use, usually as tiles and thumbnails for different flavours of rollup web parts. In the scenario described here we'll have only 8, but in some large deployments this list can grow substantially.

Let's see in which places this list of renditions is displayed, and discuss where it could make sense to get the list filtered in some way.

### 1. Rendition settings page

![Rendition settings](./rend-renditionsettings.png)

This page is at **~sitecollection/_layouts/15/ImageRenditionSettings.aspx** and only accessible by a site designer or admin, so it would make sense to give them full control on every rendition. We'll leave this as it is.

### 2. Pick rendition menu

Available to content editors by selecting an image while editing a page, then unfolding the *Pick rendition* dropdown under the contextual *Image* tab.

![Pick rendition](./rend-pickrendition.png)

These options appear sorted by the order in which they are defined in the XML file, or created from the *Rendition settings* page above, and that may not be the ideal one. This is the main place where we'll want to apply some logic to sort items so the irrelevant ones come at the bottom, or are removed altogether. 

### 3. Edit renditions page

Accessed by choosing *Edit renditions* at the bottom of the *Pick rendition* dropdown, this page displays a list of all available renditions for an image with previews of it under each rendition, allowing to change each one individually using the editor shown above.

![Edit renditions](./rend-editrenditions.png)

In this page, the images appear sorted by height **in ascending order**. This means any renditions intended for internal use such as icons in different sizes, will be displayed to users in the first place. In a future post we'll see a real life scenario where a number of such renditions can be bothering. In the solution implemented here, we'll hide these from this list.

## Proposed Solution

To tackle the problem, we'll need to address **(2)** and **(3)** above separately:

- For the *Edit renditions* page, we'll use CSS to hide the renditions we don't want to display.
- For the *Pick rendition* menu, we'll build a ribbon component which will replace the original one to introduce our custom logic to display the flyout list, while keeping unchanged the functionality when a rendition is selected.

### Identifying internal renditions

First of all we need to decide which convention we will use to separate internal renditions that will be hidden to users, and those that won't. The only data stored for a rendition is in the following snippet:
```xml
<ImageRendition>
  <Height>32</Height>
  <Id>101</Id>
  <Name>Icon small</Name>
  <Version>1</Version>
  <Width>32</Width>
</ImageRendition>
```
This leaves us two choices, both with their own pros and cons:

- Use a special convention in the `Name`, like using a **_** as a prefix.
- Use a special convention in the `Id`, such as starting with **999**.

The former is better in the sense that an admin has full control on the name when creating a new rendition via the *Rendition Settings* page, whereas IDs are assigned automatically, and can only be chosen by editing directly the **PublishingImageRenditions.xml** file in the MasterPage gallery and assigning an arbitrary `<Id/>`.

The main advantage of the latter is that the HTML table on the *Edit Renditions* page includes the ID as an attribute of the row that contains the whole control for a rendition, making it easy to target in a CSS rule.

There's not a good or bad choice here, but in this solution we'll opt for using IDs starting with **999**, just because CSS feels cleaner than hacking around with jQuery and I'm comfortable with adding any potential new *internal* rendition (which won't happen very often once a project has been deployed) to the XML file.

## Edit renditions page: Hiding items

This is what the markup in the *Edit Renditions* page looks like, once we make the IDs of our internal renditions stick to the **999** convention:

![Pick renditions](./rend-editrenditions-markup.png)

Note how the title only appears embedded within an inner node, so we would need some jQuery trick to find `<h2>` tags with a content _, then find the parent `<tr>` and hide it. Since we can't modify these pages in SharePoint Online (and we shouldn't do it in on prem either), that script would need to be embedded across the whole site collection and run in every page, after the DOM is loaded, checking whether we are in **/_layouts/15/ManageImageRenditions.aspx**. That doesn't feel very clean. 

However the css rule targeting the rendition Id would be pretty straightforward an clean:
```css
form[action*='ManageImageRenditions\.aspx'] 
  table#renditions tr[id^='rendition999'],
form[action*='ManageImageRenditions\.aspx'] 
  table#renditions tr[id^='rendition999'] + tr > td.ms-formline 
{
  display:none; 
}
```
Note how we have an extra rule for the separator line which is in every other row (SharePoint loves a good old '90s markup, you know...), and we also include a selector to target only the **ManageImageRenditions.aspx** page via the `<form>` tag. The reason for this is that the *Rendition settings* page (**ImageRenditionSettings.aspx**) has a similar markup, and we don't want to target it - Actually, if you want to hide them there as well leaving the XML file as the only point to manage internal renditions, just remove that selector.

## Custom *Pick Renditions* selector

The Ribbon in SharePoint is somewhat complex but offers great flexibility and customization opportunities, and it's quite well documented, both in [the official MSDN docs][6] and blogs like [this comprehensive series][7] from Chris O'Brien - most of it is for SharePoint 2010, but the ribbon hasn't changed much since. 

We'll need to build a custom control that overrides the default rendition picker. In particular we'll need to [replace an existing control][8], which is a [Flyout Anchor][9].

If you want to know how existing existing controls are built in order to replicate and tweak the functionality, you'll probably have to dive into the SharePoint *hive* on a SP2013 installation and try to figure out how it works, going through a scavenger hunt game which starts by finding references of the control ID. This can be found easily in the HTML markup of the ribbon component itself, in our case `Ribbon.Image.Image.Renditions.PickRendition`.

### Slight digression: Azure to the rescue

At the time of working on this solution I didn't have a SharePoint environment at hand, so I went to the [Azure portal][11], ran a search and, surprise, there are a few images of Windows Server 2012 with SP2013 and VS2013 installed. 

![Azure VM](./rend-azure-sp2013-vm.png)

I spun up a VM with all the suggested settings, and within minutes I was remoting into it. I installed Notepad++, and all ready to run a few *Find in files*'s in the `[hive]\TEMPLATE` directory for that ribbon id. 

![Find in files](./rend-azure-notepadpp.png)

This is a brute force operation which takes a while and is quite CPU intensive, usually slowing down your machine and setting the fan to spin like crazy - but in this case, it was a CPU somewhere in a datacenter, not mine. Once I was done with everything, I shut down the machine and a few pennies were added to my pay-as-you-go Azure bill. This cloud computing thing has a point.

### Overriding the existing ribbon control

Back to our research. We found a few hits of `Ribbon.Image.Image.Renditions.PickRendition`, only one of them inside at the very bottom of an XML file: `TEMPLATE\FEATURES\Publishing\ProvisionedUI.xml`. That's where the *Pick rendition* flyout is defined within a custom action.

<script src="https://gist.github.com/spcfran/2eab3574bd0655130769.js"></script>

There are 3 important bits there:

1. The `Id` attribute of the `<FlyoutAnchor>` element - We'll need to reference this value to tell SharePoint to replace this flyout with ours.
2. The `Command` attribute - This value relates to an entry in a dictionary within `SP.UI.RTE.Publishing.debug.js`, which ultimately defines the javascript function that will run when **selecting** an option from the flyout. We don't want to change this behaviour, so will leave it as it is.
3. The `PopulateQueryCommand` attribute - Again, this holds the name of a command handler in `SP.UI.RTE.Publishing.debug.js`, which in this case defines the javascript function to **populate** the menu. This is exactly what we want to replace.

So, we'll need to create our own custom action defining a similar control, but with a different query command, replacing the current one. In our case, we want to preserve the appearance, including labels and icon, but those could also be overridden. 

This is what it looks like, omitting the `<FlyoutAnchor>` attributes which are kept intact from the original one:

<script src="https://gist.github.com/spcfran/95bda549e839c55c2528.js"></script>

Note how we use original `Id` as the `Location` attribute in our `<CommandUIDefinition>` to tell SharePoint to replace the original control with ours.

Note also the that the `CommandAction` is a javascript expression calling a function (which we'll define later), and passing in whatever list of arguments are present in the context that runs this code - we'll also see what these parameters are in a minute.

### Query Command Handler in Javascript

Searching for `GetImageRenditionsMenuXml` and after a few hops within `SP.UI.RTE.Publishing.debug.js`, we get to the actual javascript functions that fills the menu with renditions:

<script src="https://gist.github.com/spcfran/eb3ae7d07cf77a6ed70a.js"></script>

Apparently the information about the available renditions already exists at runtime in a JS object called `RTE.Externals.imageRenditions.Renditions`, which holds just the id and name (with the dimensions appended to it).

![Renditions in RTE object](./rend-js-rterenditions.png)

What this code does is just format the available information as XML in a way that the ribbon framework understands, and use it to populate the `PopulationXML` property of an object that is passed into the function. 

That `properties` object will be passed to our custom function as one of the arguments as well. 

### Custom renditions menu handler in javascript

It turns out that, in `CUI.debug.js`, the Ribbon runtime actually transforms that `PopulationXML` into JSON, and assigns the result to a `PopulationJSON` property, which is the one really used to render the menu in the end. So instead of messing around with XML, in our handler we'll work with JSON directly in the format recognised by the runtime. After all it's 2016, and JSON is the cool thing now.

So here's the above function, refactored into JSON, and applying a filter function with our **999**-ID logic. You can see how easy it would be to plug in any other filtering criteria, or sorting, or any other manipulation to the `renditionItems` array you can come up with. 

<script src="https://gist.github.com/spcfran/ebf1ddcbe9e46077e128.js"></script>

Funnily enough, we got the `properties` object now as the third parameter, instead of the second as in the original command. I won't get into detail about this, but just mention that the first two parameters are irrelevant in our function (remember we are passing in the `arguments` object of the caller function, where those **are** relevant). If you are curious, you can find out more by setting breakpoints in the JS code for both the original command and our custom one, and navigating up the call stack.

An important note here: the function **is expected to return synchronously**, so take that into account when adding your custom functionality if it involves async operations. Good news is, the Ribbon runtime is smart enough and has a polling mechanism that repeatedly keeps calling your function if it doesn't populate either the `PopulationJSON` or `PopulationXML` properties, so you could easily get around this. 

## Packing everything up

We've already talked about the different parts of our solution and how we got to them. Now you can pick the parts you like to integrate in your own projects, or pick [the working solution I built for demo purposes][20], which consists of the following artifacts, all packaged up in a single Site Collection-scoped feature:

- **MO_Assets** - Module that contains and deploys the JS file implementing the custom command to the Style Library.
- **CA_FilteredPickerScriptReference** - Custom Action that inserts a reference to the above script in all your pages.
- **CA_ReplacePickRenditionButton** - Custom Action doing the actual replacement of the OOTB button with ours.
- **MO_ImageRenditions** - As a bonus for a fully working demo, a sample **PublishingImageRenditions.xml** file with some **999**ed renditions gets deployed to the master page catalog.

Note that there's no CSS file in the solution - Because the CSS rules are so simple, they have been included in the JS file and injected into the page via `document.write`, so there's no need to deploy references to the CSS file. Of course, it would be better to include it in your existing CSS files in a real world scenario, and the same goes for the JS code. In fact, doing that and leaving just the **CA_ReplacePickRenditionButton** artifact inside the feature leaves you with an easy way to revert back to the original rendition picker by just deactivating the feature.

## Summary

In this article, we:

- Introduced what image renditions are, enumerated where they are used and justified why wanted to gain control over the ones that are displayed.
- Proposed a solution by:
  - Choosing a naming convention to identify *internal* renditions to hide
  - Hiding them from pages using CSS
  - Hiding them from the *Pick Rendition* menu by creating our own ribbon control using a combination of custom actions and Javascript
- Packed all up in a SharePoint solution for Visual Studio available in [GitHub][20].

Feel free to use the solution as it is, or incorporate any part of the code in your own projects. And don't hesitate to [contribute to the repo][20] and suggest any enhancements/ideas by [raising an issue][21].



[1]: https://blog.mastykarz.nl/image-renditions-sharepoint-2013/

[6]: https://msdn.microsoft.com/en-us/library/office/ee540027(v=office.14).aspx
[7]: http://www.sharepointnutsandbolts.com/2010/01/customizing-ribbon-part-1-creating-tabs.html
[8]: https://msdn.microsoft.com/en-us/library/office/ff407619(v=office.14).aspx
[9]: https://msdn.microsoft.com/en-us/library/office/ff458372(v=office.14).aspx

[11]: https://portal.azure.com
[20]: https://github.com/spcorner/FilteredRenditionPicker
[21]: https://github.com/spcorner/FilteredRenditionPicker/issues