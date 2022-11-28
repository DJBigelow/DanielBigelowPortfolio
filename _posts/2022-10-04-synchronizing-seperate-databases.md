---
layout: single
title:  "Synchronizing Changes Between Two Separate Databases"
date:   2022-10-04 18:49:00 -0600
categories: sql postgres python
---

Around January of last year, my boss started discussing with me a new project he wanted me to work on. Snow College's student and employee portal had been showing its age and he wanted to start work on replacing it. The first part would be recreating the Snow's directory with a few added features. The most important one would be the ability for an authorized person in IT to edit employee data. Normally, this would be pretty simple, but it came with a catch. We couldn't actually change any original data in our Oracle database, which we'll call the Banner database. What my boss had in mind is that we would create a new Postgres database, which we'll call the Portal database, with a copy of the original data that we would then keep synchronized with the original database. When a change is made to either database, that authorized person from IT could then go into the directory and decide for themselves if those changes should be shown in the directory. There were a few rules as to how this would work:

- If a change is made to the Banner database and the admin decides to show the change in the directory, the corresponding entry in the Portal database is overriden
- If a change is made to the Banner database and the admin decides not to show the change in the directory, no change is made to the Portal database
- If a change is made to the Portal database and the admin decides to show the change in the directory, no change is made to the Portal database
- If a change is made to the Portal database and the admin decides to not show the change in the directory, the Portal entry is overriden with the original Banner entry
- Differences between the two databases that have been reviewed should not need to be reviewed again until there is a different change made

My original plan was to build something efficient with Pandas, a data analysis library for Python, but progress was slow. After consulting with my boss, we decided that a more straightforward and inefficient solution would be fine since Snow doesn't have a huge number of employees and the process of detecting changes between the two DBs could happen infrequently.

The solution I went with would be to read both tables of employees into memory and then to pair entries off by their employee IDs. After that, I would compare the corresponding mutable fields of each pair of employee entries and look for changes. If a change is detected and hasn't already been reviewed and tracked in a table of historical changes in the Portal database, then a pending change is added to another table in the Portal DB. These pending changes would be presented to the IT admin through the directory and their decision as to whether or not the changes get displayed would be tracked through the historical changes table I mentioned earlier. After review, pending changes are deleted.

It's maybe not a very elegant solution, but the important thing is that it was easy to do and was reasonably efficient for our purposes.

Below, I have most of the code involved with the change detection and approval/denial process. A step-by-step walkthrough would be tedious to write and read and wouldn't be all that useful either way given that I just explained how it works. I do want to talk a little bit about how my code is organized though.

I first started with a helper module filled with functions that contain the minute techincal details of how things are done, like how employee entries from both DBs are grouped, how their differences are detected, etc. I then made "service" modules that would take functions from the helper module and use them to perform more general tasks like get all unapproved changes or commit change approvals/denials. This was to make it so that to do the broader tasks, you would only have to pick out one or two functions from one or two small modules rather than pick out several functions from a large mudule and put them together. This makes the code simple at the top level and allows you to view more complex details as you drill down into it. At the top, the code looks like this: 

{% highlight python %}
router = APIRouter(
    prefix="/dir-change-detection",
    responses={404: {"description": "Not found"}}
)

@router.post("/migrate")
def start_migration():
    migrate_employees()
    return {"success": True}

@router.post("/start")
def start_change_detection():
    detect_and_commit_changes()
    return {"success": True}
{% endhighlight %}

Lower down, the code looks like this:

{% highlight javascript %}

def detect_and_commit_changes():
    banner_employees: List[Employee] = banner_employee_repository.get_banner_employees()
    portal_employees: List[Employee] = portal_employee_repository.get_portal_employees()

    grouped_employee_entries = change_detection_helper.group_banner_and_portal_employee_entries(banner_employees, portal_employees)

    all_pending_changes: List[PendingChange] = []

    for banner_employee, portal_employee in grouped_employee_entries:
        
        all_changes = change_detection_helper.get_changes_between_banner_and_portal_employee(banner_employee, portal_employee)
        historical_changes = historical_change_repository.get_historical_changes_by_badger_id(banner_employee.ID)

        employee_pending_changes = [
            pending_change for pending_change 
            in all_changes 
            if not change_detection_helper.historical_change_exists_for_pending_change(pending_change, historical_changes)
        ]

        all_pending_changes = [*all_pending_changes, *employee_pending_changes]

    pending_change_repository.insert_pending_changes(all_pending_changes)
{% endhighlight %}


And at the bottom, the code looks like this:

{% highlight python %}

def group_banner_and_portal_employee_entries(banner_employees: List[Employee], portal_employees: List[Employee]):
   
    grouped_employees = [
        (banner_emp, portal_emp)
        for banner_emp in banner_employees 
        for portal_emp in portal_employees
        if banner_emp.ID == portal_emp.ID
    ]

    return grouped_employees

def get_banner_employees_not_in_portal(banner_employees: List[Employee], portal_employees: List[Employee]):
    portal_employee_ids = [emp.ID for emp in portal_employees]
    return [banner_emp for banner_emp in banner_employees if banner_emp.ID not in portal_employee_ids]

def get_portal_employees_not_in_banner(banner_employees: List[Employee], portal_employees: List[Employee]):
    banner_employee_ids = [emp.ID for emp in banner_employees]
    return [portal_emp for portal_emp in portal_employees if portal_emp.ID not in banner_employee_ids]


def get_changes_between_banner_and_portal_employee(banner_employee: Employee, portal_employee: Employee):

    banner_emp_dict = banner_employee.dict()
    portal_emp_dict = portal_employee.dict()

    # print(banner_emp_dict)
    # print(portal_emp_dict)

    employee_columns = list(banner_employee.dict().keys())

    change_list: List[PendingChange] = []
    
    for column in employee_columns:
        if str(banner_emp_dict[column]) != str(portal_emp_dict[column]):
            print(portal_emp_dict)
            print(banner_emp_dict)

            pending_change = PendingChange(
                badger_id = banner_employee.ID,
                portal_column = column,
                banner_value = str(banner_emp_dict[column]),
                portal_value = str(portal_emp_dict[column])
            )

            change_list.append(pending_change)
    
    return change_list


def historical_change_exists_for_pending_change(pending_change: PendingChange, historical_changes: List[HistoricalChange]) -> bool:
    return any([
            historical_change_covers_pending_change(pending_change, historical_change) 
            for historical_change in historical_changes
        ]
    )
    

def historical_change_covers_pending_change(pending_change: PendingChange, historical_change: HistoricalChange) -> bool:
    return (pending_change.portal_column == historical_change.portal_column and 
            pending_change.banner_value == historical_change.banner_value and
            pending_change.portal_value == historical_change.portal_value)

{% endhighlight %}

This was an interesting problem to work on. It was interesting to have a normally straightforward task require some extra thinking and work to finish and I'm happy with the result.