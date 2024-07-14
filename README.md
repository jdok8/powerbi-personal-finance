# Personal Finance Analysis with Power BI

![Personal Finance Overview](/assets/img/expenditures_teaser.gif)

## Table of Contents

- [Introduction](#introduction)
- [Purpose of the Project](#purpose-of-the-project)
- [Features](#features)
- [Data Sources](#data-sources)
- [Usage](#usage)
- [DAX Formulas Used in Calculated Columns](#dax-formulas-used-in-calculated-columns)
- [DAX Formulas Used in Measures](#dax-formulas-used-in-measures)
- [Contributing](#contributing)
- [License](#license)
- [Authors](#authors)

## Introduction

This Power BI project enables comprehensive analysis of family expenses by consolidating data from multiple sources, including CSV files from different bank accounts. The report offers insights into account balances, income, expenses, and investment developments over time.

## Purpose of the Project

This Power BI project was created for educational purposes to demonstrate data visualization techniques in Power BI. All the data presented in the charts represent synthetic financial transactions and is in Danish Krone (DKK) unless indicated otherwise. 

## Features

- **Overview Dashboard:** Provides a summary of account balances, income, and expenses across all accounts.
  
- **Account Balance Distribution:** Visualizes the distribution of account balances for pension, current, investment, and savings accounts.

- **Income and Expenses Analysis:** Offers insights into income and expenses distribution over different time periods.

- **Expense Categorization:** Breaks down expenses by category, enabling users to identify spending patterns and areas for optimization.

- **Pension and Investment Tracking:** Tracks the performance and development of pension funds and investments over time.

## Data Sources

The project consolidates data from CSV files representing transactions from different bank accounts. New CSV files can be easily integrated to include additional transactions.

## Usage

**1. Clone the Repository**

**2. Open the Power BI Project**

Open the `.pbix` file using Power BI Desktop.

**3. Refresh Data**

If new CSV files are added with transactions, refresh the data in Power BI to update the analysis.

**4. Explore Reports**

Navigate through different pages to explore various analyses and insights provided by the report.

## DAX Formulas Used in Calculated Columns

Transaction amounts conversion from various currencies to Danish Krone (DKK) based on the currency ID and corresponding exchange rates:

    transaction_in_DKK = 
    SWITCH(
    expenditures[currency_id],
    1, expenditures[transaction] / RELATED(exchange_rates[CZK]) * RELATED(exchange_rates[DKK]),
    2, expenditures[transaction] * RELATED(exchange_rates[DKK]),
    4, expenditures[transaction] / RELATED(exchange_rates[GBP]) * RELATED(exchange_rates[DKK]),
    5, expenditures[transaction] / RELATED(exchange_rates[USD]) * RELATED(exchange_rates[DKK]), 
    6, expenditures[transaction] / RELATED(exchange_rates[BTC]) * RELATED(exchange_rates[DKK]),
    expenditures[transaction]
    )

Classifying transactions into "Internal transfer," "Initial account balance," "Expense," or "Income" based on their original type and amount:

    transaction_type = 
    IF(
    expenditures[transaction_type_org] = "Internal transfer", "Internal transfer", 
    IF(expenditures[transaction_type_org] = "Initial account balance", "Initial account balance", 
    IF(expenditures[transaction_type_org] = "Income/Expense" && expenditures[transaction] < 0, "Expense", 
    IF(expenditures[transaction_type_org] = "Income/Expense" && expenditures[transaction] > 0, "Income",
    "null"
    ))))

Creating a Date table:

    dates = CALENDARAUTO()

    year = YEAR(dates[date])

    month = FORMAT(dates[date], "mmmm")

    month_num = MONTH(dates[date])

    day = DAY(dates[date])

## DAX Formulas Used in Measures

**1. Expenses**

Expenses in absolute value:

`abs_expenses_in_DKK = ABS([expenses_in_DKK_negative_value])`

Month-Over-Month percentage change in absolute expenses represented in Danish Krone (DKK):

    abs_expenses_in_DKK MoM% = 
    VAR __PREV_MONTH = CALCULATE([abs_expenses_in_DKK], DATEADD('dates'[date], -1, MONTH))
    RETURN
	DIVIDE([abs_expenses_in_DKK] - __PREV_MONTH, __PREV_MONTH)

The average monthly expense for selected years. It computes the average expense per month by summing up the absolute transaction amounts for expenses (in Danish Krone, DKK) and then averaging them across the specified years and months:

    avg_monthly_expense_by_selected_years = 
    CALCULATE(
        AVERAGEX(
            SUMMARIZE(
                expenditures,
                dates[year],
                dates[month], 
                "Expenses",
                SUMX(
                    FILTER(expenditures, expenditures[transaction_type] = "Expense"), 
                ABS(expenditures[transaction_in_DKK])
                )
            ),
            [Expenses]
        ),
        ALLSELECTED(expenditures)
    )

Average Expense Transaction:

    avg_spent = 
    TOTALMTD(
        AVERAGEX(
            FILTER(expenditures, expenditures[transaction_type] = "Expense"),
            ABS(expenditures[transaction_in_DKK])
        ),
        dates[date]
    )

The count of unique expense transactions:

    expenses_count_of_transaction = 
    CALCULATE(
        DISTINCTCOUNT(expenditures[transcation_id]),
        expenditures[transaction_type] = "Expense"
    )

Month-Over-Month percentage change in expenses represented in negative values, while handling potential errors related to date filtering:

    expenses_in_DKK_in_negative_value_MoM% = 
    IF(
	    ISFILTERED(expenditures[date]),
	    ERROR("Time intelligence quick measures can only be grouped or filtered by the Power BI-provided date hierarchy or primary date column."),
	    VAR __PREV_MONTH =
		    CALCULATE(
			    [expenses_in_DKK_negative_value],
			    DATEADD(dates[date], -1, MONTH)
		    )
	    RETURN
		    DIVIDE([expenses_in_DKK_negative_value] - __PREV_MONTH, __PREV_MONTH)
    )

The total expenses in Danish Krone (DKK):

    expenses_in_DKK_negative_value = 
    SUMX(
        FILTER(expenditures, expenditures[transaction_type] = "Expense"), 
        expenditures[transaction_in_DKK]
    )

**2. Income**

The total income in Danish Krone (DKK):

    income_in_DKK = 
    SUMX(
        FILTER(expenditures, expenditures[transaction_type] = "Income"), 
        expenditures[transaction_in_DKK]
    )

Month-Over-Month percentage change in income, represented in Danish Krone (DKK), while handling potential errors related to date filtering:

    income_in_DKK_MoM% = 
    IF(
	    ISFILTERED(expenditures[date]),
	    ERROR("Time intelligence quick measures can only be grouped or filtered by the Power BI-provided date hierarchy or primary date column."),
	    VAR __PREV_MONTH =
		    CALCULATE(
			    [income_in_DKK],
			    DATEADD(dates[date], -1, MONTH)
		    )
	    RETURN
		    DIVIDE([income_in_DKK] - __PREV_MONTH, __PREV_MONTH)
    )

The net income in Danish Krone (DKK):

    net_income_in_DKK = [expenses_in_DKK_negative_value] + [income_in_DKK]

**3. Other**

The sum of transactions in Danish Krone (DKK) for the last month:

    sum_transaction_in_DKK_last_month = 
    TOTALMTD(
        SUM(expenditures[transaction_in_DKK]),
        expenditures[date]
    )

Year-Over-Year (YoY) change in transactions represented in Danish Krone (DKK):

    transaction_in_DKK YoY = 
    VAR __PREV_YEAR =
	    CALCULATE(
		    SUM('expenditures'[transaction_in_DKK]),
		    DATEADD('dates'[date], -1, YEAR)
	    )
    RETURN
		    SUM('expenditures'[transaction_in_DKK]) - __PREV_YEAR

Year-Over-Year (YoY) percentage change in transactions represented in Danish Krone (DKK):

    transaction_in_DKK YoY% = 
    VAR __PREV_YEAR =
	    CALCULATE(
		    SUM('expenditures'[transaction_in_DKK]),
		    DATEADD('dates'[date], -1, YEAR)
	    )
    RETURN
	    DIVIDE(
		    SUM('expenditures'[transaction_in_DKK]) - __PREV_YEAR,
		    __PREV_YEAR
	    )

The running total of transactions in Danish Krone (DKK) over time. It sums up the transaction amounts for all dates up to and including the current date:

    transaction_in_DKK_running_total = 
    CALCULATE(
	    SUM(expenditures[transaction_in_DKK]),
	    FILTER(
		    ALLSELECTED(dates[date]),
		    ISONORAFTER(dates[date], MAX(dates[date]), DESC)
	    )
    )

Retrieves the last date from the expenditures table:

    last_date = LASTDATE(expenditures[date])



## Contributing

Contributions to this project are welcome! Feel free to fork the repository, make improvements, and submit pull requests.

## License

This project is licensed under the [MIT License](LICENSE).

## Authors

- [Jan Dokoupil](https://github.com/jdok8)
