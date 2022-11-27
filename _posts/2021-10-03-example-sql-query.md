---
layout: single
title:  "A Quick Example SQL Query"
date:   2021-10-04 18:40:03 -0600
categories: sql
---

SQL is an essential tool for the common programmer. In this article, I plan on showing off what I'd consider intermediate SQL techniques.

Let's look at a simplified version of what a guitar retailer's database might look like to show off some common business logic that can be accomplished with SQL.

----------------------------------------------------------

![ERD](https://i.imgur.com/WLcORS4.png)

{% highlight sql%}
use PaymentService;

CREATE TABLE Invoice (
ID int PRIMARY KEY,
DateCreated date NOT NULL,
CustomerName varchar(64) NOT NULL,
);

CREATE TABLE Guitar (
ID int PRIMARY KEY,
GuitarName varchar(64),
Price money
);

CREATE TABLE Line (
ID int PRIMARY KEY,
InvoiceID int NOT NULL,
GuitarID int NOT NULL,
Quantity int NOT NULL,
FOREIGN KEY (InvoiceID) REFERENCES Invoice(ID),
FOREIGN KEY (GuitarID) REFERENCES Guitar(ID)
);


CREATE TABLE Payment (
ID int PRIMARY KEY,
LineID int NOT NULL,
Amount money NOT NULL
FOREIGN KEY (LineID) REFERENCES Line(ID)
);

{% endhighlight %}


----------------------------------------------------------

Here's our test data. As you can see, our store has seen some high-profile customers over the years:

----------------------------------------------------------

{% highlight sql%}
INSERT INTO Guitar VALUES (1, 'Martin 0-45', 200);
INSERT INTO Guitar VALUES (2, 'Fender Stratocaster', 500);
INSERT INTO Guitar VALUES (3, 'Ibanez Destroyer 2459', 2700);
INSERT INTO Guitar VALUES (4, '1957 Les Paul', 3800);



INSERT INTO Invoice VALUES (1, '06/15/1964', 'Bob Dylan');
INSERT INTO Line VALUES (1, 1, 1, 1); --Bob buys one Martin 0-45
INSERT INTO Payment VALUES (1, 1, 100); --Bob pays off $100 from his Martin 0-45

INSERT INTO Invoice VALUES (2, '11/20/1968', 'Jimi Hendrix');
INSERT INTO Line VALUES (2, 2, 2, 2); --Jimi buys two Fender Stratocasters 
INSERT INTO Payment VALUES (2, 2, 500); --Jimi pays for the price of one Fender


INSERT INTO Invoice VALUES (3, '01/28/1981', 'Eddie Van Halen');
INSERT INTO Line VALUES (3, 3, 3, 1); --Eddie buys one Ibanez Destroyer
INSERT INTO Line VALUES (4, 3, 4, 1);--Eddie also buys a Les Paul
INSERT INTO Payment VALUES (3, 3, 2000); --Eddie puts down $2000 on his Ibanez 
INSERT INTO Payment VALUES (4, 4, 1900); --Eddie pays for half of his Les Paul
INSERT INTO Payment VALUES (5, 4, 1900); --Eddie pays off his les Paul
{% endhighlight %}


----------------------------------------------------------

So let's say we want to look at how much each invoice totals at and how much of each one has been paid off (and maybe some other data while we're at it).
To do that, we need to do some joins in order to get the guitar associated with each line, the lines associated with each invoice, and the payments associated with each line. Let's start simple by just querying for each line's total and the amount of it that's been paid off for a single customer.

----------------------------------------------------------

{% highlight sql%}
SELECT Invoice.CustomerName,
	   Guitar.GuitarName AS Guitar, 
	   Quantity * Guitar.Price AS LineTotal, 
	   Payment.Amount AS AmountPaid 
FROM Invoice 
INNER JOIN Line ON (Line.InvoiceID = Invoice.ID)
INNER JOIN Guitar ON (Line.GuitarID = Guitar.ID)
INNER JOIN Payment ON (Payment.LineID = Line.ID)
WHERE Invoice.CustomerName = 'Eddie Van Halen';
{% endhighlight %}


| CustomerName    | LineTotal | AmountPaid |
|-----------------|-----------|------------|
| Eddie Van Halen | 2700.00   | 2000.00    |
| Eddie Van Halen | 3800.00   | 1900.00    |
| Eddie Van Halen | 3800.00   | 1900.00    |

----------------------------------------------------------

Now let's change our WHERE clause to an ORDER BY so that we can see the invoice total and the amount paid off for each customer. Since we're using aggregation, we'll have to remove the Guitar.GuitarName selection as that's not what we're grouping by and take the sum of the last two selections for the same reason.

----------------------------------------------------------

{% highlight sql%}
SELECT Invoice.CustomerName,
	   SUM(Line.Quantity * Guitar.Price) AS LineTotal, 
	   SUM(Payment.Amount) AS AmountPaid
FROM Invoice 
INNER JOIN Line ON (Line.InvoiceID = Invoice.ID)
INNER JOIN Guitar ON (Line.GuitarID = Guitar.ID)
INNER JOIN Payment ON (Payment.LineID = Line.ID)
GROUP BY CustomerName
{% endhighlight %}

| CustomerName    | InvoiceTotal | AmountPaid |
|-----------------|--------------|------------|
| Bob Dylan       | 200.00       | 100.00     |
| Eddie Van Halen | 10300.00     | 5800.00    |
| Jimi Hendrix    | 1000.00      | 500.00     |

----------------------------------------------------------

Now let's say we want to know the the amount left unpaid on each invoice. Unfortunately, we can't just subtract TotalPaid from InvoiceTotal, as you'll soon see

----------------------------------------------------------

{% highlight sql%}
SELECT Invoice.CustomerName AS CustomerName, 
		SUM(Line.Quantity * Guitar.Price) AS InvoiceTotal,
		SUM(Payment.Amount) AS TotalPaid,
		SUM(Line.Quantity * Guitar.Price) - SUM(Payment.Amount) AS TotalUnpaid
FROM Invoice
INNER JOIN Line ON (Line.InvoiceID = Invoice.ID)
INNER JOIN Guitar ON (Line.GuitarID = Guitar.ID)
INNER JOIN Payment ON (Payment.LineID = Line.ID)
GROUP BY CustomerName
{% endhighlight %}


| CustomerName    | InvoiceTotal | TotalPaid | TotalUnpaid |
|-----------------|--------------|-----------|-------------|
| Bob Dylan       | 200.00       | 100.00    | 100.00      |
| Eddie Van Halen | 10300.00     | 5800.00   | 4500.00     |
| Jimi Hendrix    | 1000.00      | 500.00    | 500.00      |

----------------------------------------------------------

That works, but that last selection seems a bit clunky. We can add a subquery to our query to potentially clean things up a bit.

----------------------------------------------------------

{% highlight sql%}
SELECT CustomerName, InvoiceTotal - TotalPaid AS RemainingBalance FROM
	(SELECT Invoice.CustomerName AS CustomerName, 
			SUM(Line.Quantity * Guitar.Price) AS InvoiceTotal,
			SUM(Payment.Amount) AS TotalPaid
		FROM Invoice
		INNER JOIN Line ON (Line.InvoiceID = Invoice.ID)
		INNER JOIN Guitar ON (Line.GuitarID = Guitar.ID)
		INNER JOIN Payment ON (Payment.LineID = Line.ID)
		GROUP BY CustomerName) AS PaymentInfo
{% endhighlight %}


----------------------------------------------------------

One other way of subquerying is with WITH AS syntax:

----------------------------------------------------------

{% highlight sql%}
WITH PaymentInfo AS
(SELECT Invoice.CustomerName AS CustomerName, 
		SUM(Line.Quantity * Guitar.Price) AS InvoiceTotal,
		SUM(Payment.Amount) AS TotalPaid
	FROM Invoice
	INNER JOIN Line ON (Line.InvoiceID = Invoice.ID)
	INNER JOIN Guitar ON (Line.GuitarID = Guitar.ID)
	INNER JOIN Payment ON (Payment.LineID = Line.ID)
	GROUP BY CustomerName)
SELECT CustomerName, InvoiceTotal - TotalPaid AS RemainingBalance FROM PaymentInfo;
{% endhighlight %}

