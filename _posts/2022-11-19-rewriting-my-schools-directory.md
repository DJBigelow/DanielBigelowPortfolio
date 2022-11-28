---
layout: single
title:  "Rewriting My College's Online Directory"
date:   2022-11-19 18:40:03 -0600
categories: javascript react python sql
---

<!-- 
For my last project workijng at Snow College's IT department, my boss had me working on rewriting Snow's online directory as the first step in creating a new student and staff portal. This wouldn't be a one-for-one recreation of the directory as the existing one had a few shortcomings and not taking the opportunity to make improvements would be a waste. 

The first and surprisingly most complex shortcoming to address was the inability for adminsitrators to easily edit records. The original directory worked off of a single authoritative source of data, which was an Oracle database. Editing this data directly was off limits (I believe doing so wouldn't fly with HR if my memory serves me correctly), so doing something as simple as displaying a faculty member's nickname instead of their given name wasn't possible. To solve this issue, I was tasked by my supervisor to create a new Postgres database that would contain a copy of the Oracle database's data. The postgres database could be updated freely, and when discrepancies between the two databases would arise, such as from records in either database being updated, an administrator would be able to use controls on the new portal to choose which changes stayed or went in the new Postgres database. For example, say that an HR employee updates the the last name of a recently married Snow College faculty member. That change goes into the Oracle database, but not the new Postgres database. At some point, a process runs that detects the change between the employee's records in the two databases. This change is then presented to some employee in IT in the admin side of the portal to either approve and bring the change from the Oracle database into the Postgres database, thus making the change visible in the new directory, or discard the change and keep the two records different. I go into the technical details of implementing this system in [this article]({{site.baseurl}}{% post_url 2022-10-04-synchronizing-seperate-databases%}). Here's how it looks in action: ![]({{site.baseurl}}/assets/img/portal/ChangeDetection.gif)

The other shortcomings included were with the search/filter controls. Entering a search query into the search bar would redirect you to a new page with the search results, which was slow and clunky, and only faculty/staff names could be used as search fields. Additionally, all faculty/staff information that was available would be displayed in a single table, which felt like an information overload. What my supervisor wanted was a table with an attached search bar that would present results in real time by filtering out results that don't match the search query. Users would see basic information on faculty/staff in the table and then be able to click on entries to see a more detailed view of the selected staff/faculty member. Administrators would also be able to click on entries and see additional information and make edits to that information.
This is what what I built looks like from the perspective of an administrator from IT: ![]({{site.baseurl}}/assets/img/portal/DirectorySearch.gif)

Here's the more detailed view of an employee: ![]({{site.baseurl}}/assets/img/portal/EmployeeDetail.png)

And here's how an admin sees that same view: ![]({{site.baseurl}}/assets/img/portal/AdminViewCensored.png)

I go into the technical details of building the directory table in [this article]({{site.baseurl}}{% post_url 2022-10-04-creating-a-table-component-in-react%}). You can also see the rest of the code for the project [here](https://github.com/DJBigelow/Capstone). Keep in mind that some parts of the project weren't written by me (mostly just the redux stuff in the frontend and the landing page) and that some code isn't included just to be safe security-wise. -->



For my last project workijng at Snow College's IT department, my boss had me working on rewriting Snow's online directory as the first step in creating a new student and staff portal. This wouldn't be a one-for-one recreation of the directory as the existing one had a few shortcomings and not taking the opportunity to make improvements would be a waste.

## Requirements

### Database Change Detection

The first and surprisingly most complex shortcoming to address was the inability for adminsitrators to easily edit records. The original directory worked off of a single authoritative source of data, which was an Oracle database (which I'll refer to as the Banner database from now on). Editing this data directly was off limits (I believe doing so wouldn't fly with HR if my memory serves me correctly), so doing something as simple as displaying a faculty member's nickname instead of their given name wasn't possible. To solve this issue, I was tasked by my supervisor to create a new Postgres database, which I'll refer to as the Portal database, that would contain a copy of the Banner database's data. The Portal database could be updated freely, and when discrepancies between the two databases would arise, such as from records in either database being updated, an administrator would be able to use controls on the new portal to choose which changes stayed or went in the new Portal database. Additionally, when a change had already been reviewed and accepted, it wouldn't need to be reviewed again. Here are the requirements summed up:

- If a change is made to the Banner database and the admin decides to show the change in the directory, the corresponding entry in the Portal database is overriden
- If a change is made to the Banner database and the admin decides not to show the change in the directory, no change is made to the Portal database
- If a change is made to the Portal database and the admin decides to show the change in the directory, no change is made to the Portal database
- If a change is made to the Portal database and the admin decides to not show the change in the directory, the Portal entry is overriden with the original Banner entry
- Differences between the two databases that have been reviewed should not need to be reviewed again until there is a different change made

### Online Directory

There were also issues with Snow's original online directory. Entering a search query into the search bar would redirect you to a new page with the search results, which was slow and clunky, and only faculty/staff names could be used as search fields. Additionally, all faculty/staff information that was available would be displayed in a single table, which felt like an information overload. What my supervisor wanted was a table with an attached search bar that would present results in real time by filtering out results that don't match the search query. Users would see basic information on faculty/staff in the table and then be able to click on entries to see a more detailed view of the selected staff/faculty member. Administrators would also be able to click on entries and see additional information and make edits to that information. Here are the user stories my supervisor and I agreed upon for the new online directory:

#### Actors

- Standard User
- Admin User

#### User Stories

##### Standard User
- As a user I want to quickly search for an employee and find their phone number
- As a user, I can find people by department
- As a user, I can find people by position
- As a user, I can copy someone's email address to my clipboard
- As a user, I want my search to update on each keystroke rather than when I enter my query
- As a user, when I click on a name, I can see that person's details (department, room number, position, etc.)
- As a user, I must authenticate before viewing the directory
- As a user, I can filter by campus
- As a user, I can see contact information for departments themselves
- As a user, I want the directory to be fast

##### Admin User
- As an admin, I can modify all information on the directory
- As an admin, I am prompted to approve pending changes
- As an admin, I can hide and show users on the directory (stretch goal)

## Solutions

### Database Change Detection

I'll only give a high-level overview of how I solved this problem here. If you want a more slightly more in-depth look at the technical details, I give one in [this article]({{site.baseurl}}{% post_url 2022-10-04-synchronizing-seperate-databases%}). 

First, I created a Python + FastAPI server with an endpoint that would trigger the change detection process when it was hit. When the change detection process started, it would query all employees from both databases and group the employee records by their Snow IDs. The process would then compare each column/attribute of each pair of employee records with each other to find any differences. When differences were found, they would be inserted into a table labeled "pending_change" in the Portal database, but not if the change had already been approved and tracked in a Portal database table labeled "historical_change". From there, an admin could review any pending changes through the new portal and approve/discard them. Changes that were approved would be tracked in the "historical_change" table to prevent them from needing to be reviewed again, like I mentioned before. Approving a change would mean that no data in the Portal DB would get changed. Discarding a change would mean that the differing column/attribute of the Portal employee would be overridden by the value of the column/attribute of the corresponding Banner DB entry.

Here's an activity diagram of the process: 

![]({{site.baseurl}}/assets/img/portal/change_detection_activity.png)

Here's what the process of approving/discarding changes looks like:

![]({{site.baseurl}}/assets/img/portal/ChangeDetection.gif)



### Online Directory

I used React Table, a library of React hooks for building tables to create a table component with filter/search controls. You can see how I did that in [this article]({{site.baseurl}}{% post_url 2022-10-04-creating-a-table-component-in-react%}). Here's what it looks like in action:

![]({{site.baseurl}}/assets/img/portal/DirectorySearch.gif)

Here's the detail view of an employee, complete with controls to copy the viewed employee's email to the clipboard:

![]({{site.baseurl}}/assets/img/portal/EmployeeDetail.png)

And here's the code for displaying that information:

{% highlight javascript %}
import { useState } from "react";
import Employee from "../models/Employee";

type EmployeeCardProps = {
  employee: Employee;
  admin: boolean;
};

export const EmployeeCard = ({ employee, admin }: EmployeeCardProps) => {
  const [emailCopied, setEmailCopied] = useState(false);

  return (
    <div className="card">
      <div className="card-title mb-3 ps-3">
        <h4 className="display-4">{employee.fullDisplayName}</h4>
      </div>

      <div className="card-body pt-0">
        {/* Email + copy button */}
        <div className="row d-flex justify-content-start align-items-start">
          
          {admin && (
            <div>

              <div className="row">
                <p>
                  <b>Badger ID:</b> {employee.id}
                </p>
              </div>

              <div className="row">
                <p>
                  <b>Real First Name:</b> {employee.realFirstName}
                </p>
              </div>

              <div>
                <p>
                  <b>Preferred First Name:</b> {employee.preferredFirstName}
                </p>
              </div>
            </div>
          )}

          <p className="col-auto">
            <b>Email:</b> {`${employee.snowEmail}`}
          </p>
          <button
            disabled={
              employee.snowEmail === undefined ||
              employee.snowEmail === Employee.undefinedPropertyFallback
            }
            className="btn btn-primary btn-sm col-auto"
            onClick={() => {
              navigator.clipboard.writeText(employee.snowEmail as string);
              setEmailCopied(true);
            }}
          >
            <i
              className={
                emailCopied ? "bi bi-clipboard-check" : "bi bi-clipboard"
              }
            />
          </button>
        </div>

        <div className="row">
          <p>
            <b>Department:</b> {employee.department}
          </p>
        </div>

        <div className="row">
          <p>
            <b>Position:</b> {employee.position}
          </p>
        </div>

        <div className="row">
          <p>
            <b>Campus:</b> {employee.campus}
          </p>
        </div>

        <div className="row">
          <p>
            <b>Building:</b> {employee.building}
          </p>
        </div>

        <div className="row">
          <p>
            <b>Office Room Number:</b> {employee.roomNumber}
          </p>
        </div>

        <div className="row">
          <p>
            <b>Office Phone Number:</b> {employee.fullDisplayOfficePhone}
          </p>
        </div>
      </div>
    </div>
  );
};

{% endhighlight %}

Administrators get a different view of the table:

![]({{site.baseurl}}/assets/img/portal/AdminViewCensored.png)

Admins also get to see more information in the detail view of an employee.

![]({{site.baseurl}}/assets/img/portal/AdminDetailCensored.png)

They also have a button they can click that brings up a form for editing the employee being viewed:

![]({{site.baseurl}}/assets/img/portal/EditValidation.png)

Here's the code for the form for editing an employee's information. Most of the form components weren't written by me, so I won't be showing them off:

{% highlight javascript %}
import { TextInputRow, useTextInput } from '../../../../components/forms/TextInputRow'
import { SelectInputRow, useSelectInput } from '../../../../components/forms/SelectInputRow';
import { LoadingSpinner } from '../../../../components/LoadingSpinner';
import Employee from '../../shared/models/Employee'

import { usePhoneNumberInput, PhoneNumberInputRow } from '../../../../components/forms/phoneNumberInputRow';

import { useBuildingsQuery } from '../../shared/services/buildingService';
import { useDepartmentsQuery } from '../../shared/services/departmentsService';
import  { FormEvent, useState } from 'react';
import { useUpdateEmployeeMutation } from '../../shared/services/employeeService';
import { addAlert, alertOnError } from '../../../../store/alertSlice';
import { useStoreDispatch } from '../../../../store';


type EmployeeEditFormProps = {
    employee: Employee;
}

export const EmployeeEditForm = ({ employee }: EmployeeEditFormProps) => {
    const dispatch = useStoreDispatch();
    const employeeUpdateMutation = useUpdateEmployeeMutation()

    const realFirstNameControl = useTextInput(employee.realFirstName, {max: 60, required: true})
    const preferredFirstNameControl = useTextInput(employee.preferredFirstName || '', {max: 60, required: false})
    const middleNameControl = useTextInput(employee.middleName || '', {max: 60, required: false})
    const lastNameControl = useTextInput(employee.lastName, {max: 240, required: true})
    const snowEmailControl = useTextInput(employee.snowEmail || '', {max: 128, required: false})
    const positionControl = useTextInput(employee.position || '', {max: 120, required: true})

    const roomNumberControl = useTextInput(employee.roomNumber || '', {max: 12, required: false})


    const selectCampusControl = useSelectInput<string>({
        initialValue: "Ephraim",
        options: ["Ephraim", "Richfield"],
        getKey: (campus: string) => campus,

    })

    const buildingsQueryResult = useBuildingsQuery();
    const departmentsQueryResult = useDepartmentsQuery();


    const selectBuildingControl = useSelectInput<string>({
        options: (buildingsQueryResult.isLoading || buildingsQueryResult.isError) ? [] : buildingsQueryResult.data as string[], 
        getKey: (building: string) => building,
    })

    const selectDepartmentControl = useSelectInput<string>({
        options: (departmentsQueryResult.isLoading || departmentsQueryResult.isError) ? [] : departmentsQueryResult.data as string[],
        getKey: (department: string) => department 
    })


    const phoneNumberInputControl = usePhoneNumberInput(employee.fullDisplayOfficePhone || '')


    const formHasErrors = () => {
        return !!realFirstNameControl.error ||
               !!lastNameControl.error ||
               !!positionControl.error ||
               !!phoneNumberInputControl.error

    }

    const submitHandler = (e: FormEvent) => {
        e.preventDefault();

        if (formHasErrors()) {
            dispatch(addAlert("Form has errors"))
        }
        else {

            const updatedEmployee = {
                ID: employee.id,
                FIRST_NAME: realFirstNameControl.value,
                LAST_NAME: lastNameControl.value,
                PREF_NAME: preferredFirstNameControl.value,
                MI: middleNameControl.value,
                SNOW_EMAIL: snowEmailControl.value,
                POSITION: positionControl.value,
                DEPT_DESC: selectBuildingControl.value,
                CAMPUS: selectCampusControl.value,
                BUILDING_DESC: selectBuildingControl.value,
                OFFICE_ROOM: roomNumberControl.value,
            }

            console.debug("updatedEmployee", updatedEmployee)

            employeeUpdateMutation.mutate(updatedEmployee)
        }

    }

    if (buildingsQueryResult.isLoading || buildingsQueryResult.isIdle || departmentsQueryResult.isLoading || departmentsQueryResult.isIdle) {
        return (
            <div className='d-flex justify-content-center align-items-center'>
                <LoadingSpinner />
            </div>
        )
    }

    if (buildingsQueryResult.isError || departmentsQueryResult.isError) {
        return (
            <div>
                {
                    buildingsQueryResult.isError &&
                    <h1>Error fetching buildings</h1>
                }
                {
                    departmentsQueryResult.isError &&
                    <h1>Error fetching deparments</h1>
                }
            </div>
        )
    }
   
    

    return (
        <div>
            <div className='container'>
                <form onSubmit={submitHandler}>
                   
                    <TextInputRow label='First Name' control={realFirstNameControl} />
                    <TextInputRow label='Preferred First Name' control={preferredFirstNameControl} />
                    <TextInputRow label='Middle Name' control={middleNameControl} />
                    <TextInputRow label='Last Name' control={lastNameControl} />
                    <TextInputRow label='Snow Email' control={snowEmailControl} />
                    <TextInputRow label='Position' control={positionControl} />
                    <TextInputRow label='Office Room Number' control={roomNumberControl} />

                    <SelectInputRow label='Campus' control={selectCampusControl} />
                    <SelectInputRow label='Building' control={selectBuildingControl} />
                    <SelectInputRow label='Department' control={selectDepartmentControl} />

                    <PhoneNumberInputRow label='Office Phone' control={phoneNumberInputControl} />

                    <div className='form-group row mt-2'>
                        <div className='col-md-2 my-auto' />
                        <div className='col-md-1 my-auto'>
                            <button type='submit' className='btn btn-primary'>Save</button>

                        </div>
                        <div className='col-md-2 my-auto'>
                            <button type='reset' className='btn btn-danger'>Discard Changes</button>
                        </div>
                    </div>
                    
                </form>
            </div>
        
        </div>
    )
}
{% endhighlight %}

If you're curious about what's up with the QueryResult checks or the useUpdateEmployeeMutation hook, check out [this article]({{site.baseurl}}{% post_url 2022-10-04-using-react-query-to-manage-server-state%}). You can also see the rest of the code for the project [here](https://github.com/DJBigelow/Capstone). Keep in mind that some parts of the project weren't written by me (mostly just the redux stuff in the frontend and the landing page) and that some code isn't included just to be safe security-wise.



