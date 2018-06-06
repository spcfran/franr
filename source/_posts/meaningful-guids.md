---
title: Meaningful GUIDs
permalink: meaningful-guids
date: 2016-01-04
tags:
excerpt: We use GUIDs every day to uniquely identify a variety of elements in our applications. They are a meaningless chunk of letters and digits which don't give us a clue of what they represent. Or, do they?
---
We use GUIDs every day to uniquely identify a variety of elements in our applications. They are a meaningless chunk of letters and digits which don't give us a clue of what they represent. Or, do they?
<!-- less -->

To put it simply, GUIDs (or UUIDs) are 128-bit numbers typically used as unique identifiers of things. They are usually represented as strings of 128/4=32 hexadecimal characters separated by dashes in groups of 8-4-4-4-12. Sometimes the a-f letters are uppercase and some lowercase, and sometimes they also appear enclosed in curly braces. The following 2 sample GUIDs represent the same number, and any good parser should account for the variances and treat them as the same thing.
```
 de305d54-75b4-431b-adb2-eb6b9e546014
{DE305D54-75B4-431B-ADB2-EB6B9E546014}
```
The point of being so long is that you can generate a GUID randomly and there's an [astronomical chance](https://en.wikipedia.org/wiki/Universally_unique_identifier#Random_UUID_probability_of_duplicates) of it being unique in the world. And we don't even need it to be unique in the world to avoid clashes, only in the domain of whatever we are doing. 

This is very useful because it saves us from keeping a centralized authoritative database of IDs to ensure unicity, and perhaps more importantly, making a round trip to the store every time we need to request a new one.

## GUIDs in SharePoint

GUIDs are used everywhere in SharePoint. They globally identify pretty much everything from fields, content types, features, lists, items, term store artifacts, workflows, and a long etc. They are immutable and well-known so for example, if you google `fa564e0f-0c70-4ab9-b863-0177e6ddd247`, the id of the **Title** field, you'll get a few results.

Now take for example [this list of SharePoint out of the box features](http://blogs.msdn.com/b/mcsnoiwb/archive/2010/01/07/features-and-their-guid-s-in-sp2010.aspx) and search for `00bfea71` in the page. There are 24 occurrences, meaning that at least that many SharePoint internal GUIDs contain (actually, begin with) that sequence of 8 hexadecimal digits. What are the odds of 24 randomly generated GUIDs beginning with the same 32 bits? Very low, for sure.

The SharePoint guys were a bit naughty here (or should I say clever?) and didn't stick to the rule that [version 4 GUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29) should be generated in a *proper* [pseudo-]random way, in favour of some readability. They used some [Hexspeak](https://en.wikipedia.org/wiki/Hexspeak) tricks to make their sea of GUIDs much more navigable.

Let's see some examples of these OOB feature IDs from the above list:
	
- This OOB feature deploys the *Announcements* list template, which has a [ServerTemplateId](https://msdn.microsoft.com/EN-US/library/office/microsoft.sharepoint.splisttemplatetype.aspx) of 104:
        
    ```
    00bfea71-d1ce-42de-9c63-a44004ce0104
    OOBFEAT                 ANNOONCE 104
    ```
- This one provisions some *basic web parts*:

    ```
    00bfea71-1c5e-4a24-b310-ba51c3eb7a57
    OOBFEAT                 BASICWEBPART
    ```
- This one deploys the *Data Connections* Library template, with a Template ID of 130:

    ```
    00bfea71-dbd7-4f72-b8cb-da7ac0440130
    OOBFEAT                 DATACONN 130
    ```

You get the picture. Go ahead and try to find some more patterns in the rest of the `00bfea71` GUIDs in the list.

## Use it

To use this technique, simply generate a random GUID and replace the characters as required, keeping the rest as generated to introduce some randomness.

Agree a set of rules within your team, and decide where (i.e. for which artifacts) to use it. Some tips:

- Hexspeak is not a standard, choose your variant: how will you represent letters for which there's not an obvious number. Define your own *mapping*.
- Choose a prefix to use as **namespace** and identify your project/company.
- A character could be also placed somewhere to identify the **artifact type**. For example, `C` for Columns, `F` for features, or `7` for Term Sets. If you place this next to the project prefix, it could be an easy way to find all references to components of a certain type by searching for e.g. `F5A45C03-C`.

> Warning: Not all the bits in a GUID are meant to be random. Check the [Version 4 section in the Wikipedia article](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_.28random.29) to see which ones are fixed.

## Advantages

What's the point of doing this? Here are some cases where I found it useful:

- When reading code and stumbling upon a GUID, you can identify what it represents without needing to check any documentation or search where it was defined.
- Search by namespace (and artifact type) to identify all your own GUIDs, e.g. within a SharePoint ULS log or doing a solution-wide search.
- If your Hexspeak-based conventions are strong enough, you can search for the exact descriptor of the GUID, and not just the namespace.

## Randomness implications

The downside to the idea described is that, by fixing a part to a "namespace" and arbitrarily setting other to the meaningful word/phrase that describes the GUID, the bits that remain actually random are greatly reduced, thus increasing the probability of a collision. This could be truly a risk, however bear in mind that:

- The namespace/prefix is somehow random on its own. It is repeated across your own GUIDs, but you have control over these. The same applies to the description/suffix.
- The GUIDs only need to be unique within your project (or within the SharePoint world if you are developing some reusable product or library), and that's a small domain, so I think it is pretty safe to assume the loss in randomness.

Now it's up to you. Will you be a Math purist (or just not care!) and keep using random GUIDs, or take the practical path? Feel free to 1EABE-A-C033E4T with your views!
