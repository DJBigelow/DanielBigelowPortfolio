---
layout: single
title:  "Rewriting My College's Online Directory"
date:   2022-11-19 18:40:03 -0600
categories: javascript react python sql
---

For my last project workijng at Snow College's IT department, my boss had me working on rewriting Snow's online directory as the first step in creating a new student and staff portal. This wouldn't be a one-for-one recreation of the directory as the existing one had a few shortcomings and not taking the opportunity to make improvements would be a waste. 

The first and surprisingly most complex shortcoming to address was the inability for adminsitrators to easily edit records. The original directory worked off of a single authoritative source of data, which was an Oracle database. Editing this data directly was off limits (I believe doing so wouldn't fly with HR if my memory serves me correctly), so doing something as simple as displaying a faculty member's nickname instead of their given name wasn't possible. To solve this issue, I was tasked by my supervisor to create a new Postgres database that would contain a copy of the Oracle database's data. The postgres database could be updated freely, and when discrepancies between the two databases would arise, such as from records in either database being updated, an administrator would be able to use controls on the new portal to choose which changes stayed or went in the new Postgres database. For example, say that an HR employee updates the the last name of a recently married Snow College faculty member. That change goes into the Oracle database, but not the new Postgres database. At some point, a process runs that detects the change between the employee's records in the two databases. This change is then presented to some employee in IT in the admin side of the portal to either approve and bring the change from the Oracle database into the Postgres database, thus making the change visible in the new directory, or discard the change and keep the two records different. I go into the technical details of implementing this system in [this article]({{site.baseurl}}{% post_url 2022-10-04-synchronizing-seperate-databases%}). Here's how it looks in action: ![]({{site.baseurl}}/assets/img/portal/ChangeDetection.gif)

The other shortcomings included were with the search/filter controls. Entering a search query into the search bar would redirect you to a new page with the search results, which was slow and clunky, and only faculty/staff names could be used as search fields. Additionally, all faculty/staff information that was available would be displayed in a single table, which felt like an information overload. What my supervisor wanted was a table with an attached search bar that would present results in real time by filtering out results that don't match the search query. Users would see basic information on faculty/staff in the table and then be able to click on entries to see a more detailed view of the selected staff/faculty member. Administrators would also be able to click on entries and see additional information and make edits to that information.
This is what what I built looks like from the perspective of an administrator from IT: ![]({{site.baseurl}}/assets/img/portal/DirectorySearch.gif)

Here's the more detailed view of an employee: ![]({{site.baseurl}}/assets/img/portal/EmployeeDetail.png)

And here's how an admin sees that same view: ![]({{site.baseurl}}/assets/img/portal/AdminViewCensored.png)

I go into the technical details of building the directory table in [this article]({{site.baseurl}}{% post_url 2022-10-04-creating-a-table-component-in-react%}). You can also see the rest of the code for the project [here](https://github.com/DJBigelow/Capstone). Keep in mind that some parts of the project weren't written by me (mostly just the redux stuff in the frontend and the landing page) and that some code isn't included just to be safe security-wise.


