---
layout: single
title:  "Managing Remote Data with React-Query"
date:   2022-10-04 19:00:00 -0600
categories: react javascript web-dev
---

While I was working on Snow College's new student portal, I had to find a way to manage data recieved from the server in my client. I had done this before in Vue, but it had been decided that we would be using React for new projects going forwards, so I had to figure out how to do things the React way. I had worked with Redux a little in the past and knew that was the most common way of managing application state in React, so that's where I started. What I found out is that Redux is good for your own application's state, but not quite as well suited for keeping data from another source up to date. It's important to keep in mind that when a client application recieves data from a server, that data doesn't really belong to the client. 

There's a programming joke that goes "What are the two hardest things in programming? Naming things, cacheing, and off-by-one errors". When the client application holds on to data from a server, it's essentially cacheing it. This cached data can go bad and needs to be refreshed from time to time. It's important that communication with the server is done asynchronously so that your user interface doesn't freeze every time a request is made, which is something I found Redux isn't great for. It can be done, but it takes a bit of work. 

Before I went though with Redux for managing server state, my boss recommended a package called React-Query, which describes itself as "Hooks for managing, caching and syncing asynchronous and remote data in React". I think looking at React Query's usage will help me explain what it's specifically used for and why it does it well, so let's take a look at some of the code I wrote that utilizes it:


{% highlight javascript %}
const queryClient

const employeesKey = "Employees"

export const useAllEmployeesQuery = () => {

    return useQuery<Employee[]>(employeesKey, async () => {
        const url = '/api/employees/get-all'
        const options = {
            headers: await getAuthHeader()
        }

        const response = await axios.get(url, options)

        if (response.data.success === false) {
            throw Error(`Error getting employees from API: ${response.data.message}`)
        }

        return response.data.employees.map((employee: Employee) => formatResponseEmployeeData(employee))
    });
}

export const useUpdateEmployeeMutation = () => {
    return useMutation(async (employee: any) => {
        const url = `/api/employees/update`
        const options = { headers: await getAuthHeader()}

        console.debug("mutation employee", employee)
        const response = await axios.post(url, { ...employee })

        if (response.data.success === false) {
            throw Error(`Error updating employee: ${response.data.message}`);
        }
    }, 
    { onSuccess: () => queryClient.invalidateQueries(employeesKey) });
}

{% endhighlight %}

These functions are used to get a list of employees and update individual employees through the FastAPI server I wrote for this project. Notice the "employeesKey" constant on line 3 and its usage at the beginning of the first function and end of the last function. 
React Query allows you to write code that is run when a query to some remote data source is made. The data that is recieved from that source is cached. That cache needs some sort of handle, so the user defines one that React Query will use to identify said cache going forwards. That's what "employeesKey" is. In the "useAllEmployeesQuery" function, I'm defining how the list of employees I mentioned earlier is queried and I'm giving the cache those employees are stored in when they're recieved the key "Employees". Pretty simple and not all that groundbreaking.

The cool part, and the part that makes React Query so useful, is mutations, which I set up in the "useUpdateEmployeeMutation" function. Here, I set up a web request that sends updated employee information to the backend. Notice the last line of the function. This line says that if the request runs without any errors, invalidate the cache with the handle of "employeesKey (i.e. Employees)". When you tell React Query to invalidate its cache, it'll refresh it using a query you define yourself. In the case of my code, the query set up in "useAllEmployeesQuery" is run again and used to refresh the cache.

I found this solution to be much easier than using Redux. I ended up writing a fraction of the code I would've had to otherwise, and everything is fairly robust.

Before we wrap things up, let's take a quick look at how React Query is used by a component and how simple it is:

{% highlight javascript %}
const employeeQueryResult = useAllEmployeesQuery()

    if (employeeQueryResult.isLoading || employeeQueryResult.isIdle) {
        return ( 
            <div className='d-flex justify-content-center align-items-center'>
                <LoadingSpinner /> 
            </div>
        )
    }
    
    if (employeeQueryResult.isError) {
        return ( <div><h1>{`Error fetching employees: ${employeeQueryResult.error}`}</h1></div> )
    }

    
    return (
        <div className='container'>
            <h1 className='text-center display-3'>Snow College Directory</h1>

            <Table columns={columns}
                   data={employeeQueryResult.data} 
                   hiddenColumns={hiddenColumns}
                   rowClick={navToDetail}
                   filterPlaceholderText='Search by name, department, building, etc.'/>
        </div>
    )
{% endhighlight %}


You can see here that you can define what your component looks like while your query is running, when your query fails, and when your query is successful. Pretty neat.