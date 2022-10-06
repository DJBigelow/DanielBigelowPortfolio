---
layout: single
title:  "Creating a table component in React"
date:   2022-10-04 18:40:03 -0600
categories: jekyll update
---
I recently worked on revamping Snow College's online directory. We had just switched from using Node.js and Vue to React and FastAPI and I needed to work out how to make a good reusable table component that could be used in my project along with other interns' projects. Since tables are a pretty common UI element, I first checked to see if there were any good packages that were being continuously maintained that I could leverage for my own needs so that I didn't end up reinventing the wheel. The most promising option I found was React Table, a set of hooks you can use to set up table component. The nice thing about React Table is that it gives you full control over how your tables are rendered, making them configurable for different situations. React Table also has support for efficient filtering, which really came in handy for this project.


Let's take a look through the table component I built to see how React Table is used, starting with the props:

{% highlight js%}

import { useTable, useGlobalFilter, useFilters, Row } from "react-table";
import { TableFilter } from "./TableFilter";

type TableProps<T> = {
  columns: {
    Header: string;
    accessor: string | ((originalRowObj: T, rowIndex: number) => any);
    Cell?: ({ value }: any) => string | React.Component
  }[];
  data: T[];
  hiddenColumns?: string[];
  rowClick?: any;
  filterPlaceholderText?: string;
};

{% endhighlight %}


So we have "columns", an object with three properties, "Header", "accessor" and "Cell". "Header", as you can guess, is just a string specifying the header of a column. "accessor" can be a string or function that is used to access the property of a row object that falls under the given column in question. If a string is supplied for this, React Table tries to access a property with that key. If you supply a function, the row object is passed to it along with the index of the row, then it's up to the function how the access is made. Lastly is the "Cell" property. Cell is simply a function used for formatting cells, which I ended up using to automatically provide fallback values for empty cells.

After that we have "data", which is just the array that populates your table. "hiddenColumns" lets you name specific columns that shouldn't be displayed but can be used for filtering. "rowClick" is something I included, which is a callback function that is invoked when a row is clicked (I don't remember why its type is "any"). "filterPlaceholderText" is included by me and is does exactly as the name implies.

The following section of code is mostly just React Table boilerplate, with exception to the inclusion of "useFilters" and "useGlobalFilter" as arguments for the "useTable" hook. Those are extensions for React Table that allows you to filter entries in your table that I included.

{% highlight js %}
export const Table = <T extends unknown>({
  columns,
  data,
  hiddenColumns,
  rowClick,
  filterPlaceholderText,
}: TableProps<T>) => {
  const {
    getTableProps,
    getTableBodyProps,
    headerGroups,
    rows,
    prepareRow,
    state,
    setGlobalFilter,
  } = useTable(
    {
      columns: columns as any,
      data: data as object[],
      initialState: { hiddenColumns: hiddenColumns || [] },
    },
    useFilters,
    useGlobalFilter
  );
```
{% endhighlight %}

Now comes the component itself. At the top you'll notice a "TableFilter" child component, which I'll get into later. The rest of the code is just usage of React Table as the documentation recommends, with exception of the onClick event handler for the `<tr>` tag, which uses the "rowClick" callback function I talked about earlier. It should be mentioned that React Table does have support for selecting rows, but as far as I can tell, you can only toggle the selection of multiple rows rather than run code when a single row is clicked, which is why I added the whole "rowClick" business to the component. 

{% highlight js %}

  return (
    <div>
      <TableFilter
        globalFilter={state.globalFilter}
        setGlobalFilter={setGlobalFilter}
        placeHolderText={filterPlaceholderText}
      ></TableFilter>

      <div className="table-responsive">
        <table
          {...getTableProps()}
          className="table table-hover table-striped"
          style={{ height: "400px" }}
        >
          <thead>
            {headerGroups.map((headerGroup) => (
              <tr {...headerGroup.getHeaderGroupProps()}>
                {headerGroup.headers.map((column) => (
                  <th {...column.getHeaderProps()} scope="col">
                    {column.render("Header")}
                  </th>
                ))}
              </tr>
            ))}
          </thead>
          <tbody {...getTableBodyProps()}>
            {rows.map((row: Row<object>) => {
              prepareRow(row);
              return (
                <tr
                  {...row.getRowProps()}
                  onClick={() => {
                    // If a rowClick function was included, call it with the row's internal data as an argrument
                    if (rowClick) rowClick(row.original as T);
                  }}
                  // Enable pointer cursor if row selection is enabled
                  style={{ cursor: rowClick ? "pointer" : "auto" }}
                >
                  {row.cells.map((cell) => (
                    <td {...cell.getCellProps()}>{cell.render("Cell")}</td>
                  ))}
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
};
{% endhighlight %}

Let's take a look at the TableFilter component I mentioned earlier:

For the most part, this component simply manages the state of a search-bar-like element and adds asynchronous debouncing to inputs so that the table isn't updated by the filter until the user stops typing. There's not much to look at here, which is fantastic. Being able to efficiently filter a collection of data in your user interface without writing a large amount of code is a great thing.

{% highlight js %}
import { useAsyncDebounce } from "react-table";

import { useState } from "react";

export const TableFilter = ({ globalFilter, setGlobalFilter, placeHolderText }: any) => {
  const [filterValue, setFilterValue] = useState(globalFilter);

  const onFilterChange = useAsyncDebounce((filterValue) => {
    setGlobalFilter(filterValue || undefined);
  }, 200);

  return (
    <input
      placeholder={placeHolderText}
      value={filterValue || ""}
      onChange={(e) => {
        setFilterValue(e.target.value);
        onFilterChange(e.target.value);
      }}
      className="form-control"
    ></input>
  );
};
{% endhighlight %}

Finally, let's look at how the table component is used:

{% highlight js %}
export const Directory = () => {

    const columns = [

        {
            Header: 'Name',
            accessor: (employee: Employee, _rowIndex: any) => employee.fullDisplayName,
        },
        {
            Header: 'Email',
            accessor: (employee: Employee, _rowIndex: any) => employee.snowEmail,
            Cell: ({ cell: { value }}: any) => value || Employee.undefinedPropertyFallback
        },
        {
            Header: 'Position',
            accessor: (employee: Employee, _rowIndex: any) => employee.position,
            Cell: ({ cell: { value }}: any) => value || Employee.undefinedPropertyFallback
        },
        {
            Header: 'Department',
            accessor: (employee: Employee, _rowIndex: any) => employee.department,
            Cell: ({ cell: { value }}: any) => value || Employee.undefinedPropertyFallback
        },
        {
            Header: 'Campus',
            accessor: (employee: Employee, _rowIndex: any) => employee.campus,
            Cell: ({ cell: { value }}: any) => value || Employee.undefinedPropertyFallback
        },
        {
            Header: 'Office Building',
            accessor: (employee: Employee, _rowIndex: any) => employee.building,
            Cell: ({ cell: { value }}: any) => value || Employee.undefinedPropertyFallback
        },
    ]

    const hiddenColumns = [
        'Department',
        'Office Building',
    ]

    ...
    ...
    ...

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

}
{% endhighlight %}

Pretty straightforward. What I have here is a table of employees where individual employees can be inspected in detail by clicking on them. Records can be filtered by any column, and the "Department" and "Office Building" columns are hidden to remove clutter. I enjoyed being able to provide accessors for columns, which can help safeguard you from making typos when specifying properties to be accessed, or prevent you from trying to access a property that has been changed or removed. I also liked just how easy it was to add powerful filter functionality to the table. All in all, React Table turned out to be a great choice.

