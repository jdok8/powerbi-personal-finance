# Personal Finance Analysis with Power BI

![Personal Finance Overview](/assets/img/expenditure_teaser.gif)

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

This Power BI project enables comprehensive analysis of family expenses by consolidating data from multiple sources, including CSV and XLSX files from different bank accounts. The report offers insights into account balances, income, expenses, and investment developments over time.

## Purpose of the Project

This Power BI project was created for educational purposes to demonstrate data visualization techniques in Power BI. All the data presented in the charts represent synthetic financial transactions and is in Danish Krone (DKK) unless indicated otherwise. 

## Features

- **Overview Dashboard:** Provides a summary of account balances, income, and expenses across all accounts.
  
- **Account Balance Distribution:** Visualizes the distribution of account balances for pension, current, investment, and savings accounts.

- **Income and Expenses Analysis:** Offers insights into income and expenses distribution over different time periods.

- **Expense Categorization:** Breaks down expenses by category, enabling users to identify spending patterns and areas for optimization.

- **Pension and Investment Tracking:** Tracks the performance and development of pension funds and investments over time.

## Data Sources

The project consolidates data from CSV and XLSX files representing transactions from different bank accounts. New files can be easily integrated to include additional transactions.

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

    transaction_DKK = 
    SWITCH(
	    transactions[currency_id],
	    2, transactions[transaction] / RELATED(exchange_rates[CZK]) * RELATED(exchange_rates[DKK]),
	    3, transactions[transaction] / RELATED(exchange_rates[USD]) * RELATED(exchange_rates[DKK]), 
	    4, transactions[transaction] * RELATED(exchange_rates[DKK]),
	    5, transactions[transaction] / RELATED(exchange_rates[ETH]) * RELATED(exchange_rates[DKK]),
	    6, transactions[transaction] / RELATED(exchange_rates[BTC]) * RELATED(exchange_rates[DKK]),
	    7, transactions[transaction] / RELATED(exchange_rates[GBP]) * RELATED(exchange_rates[DKK]),
	    transactions[transaction]
    )

Classifying transactions into "Internal transfer," "Initial account balance," "Expense," or "Income" based on their original type and amount:

    transaction_type = 
    IF(
	    transactions[transaction_type_unsorted] = "Vnitřní převod", "internal transfer",
	    IF(transactions[description] = "Vyrovnání stavu na účtu", "internal transfer", 
	    IF(transactions[transaction_type_unsorted] = "Počáteční stav účtu", "initial account balance", 
	    IF(transactions[transaction_type_unsorted] = "Příjem/Výdaj" && transactions[transaction] < 0, "expense", 
	    IF(transactions[transaction_type_unsorted] = "Příjem/Výdaj" && transactions[transaction] > 0, "income",
	    IF(transactions[transaction_type_unsorted] = "Výdaj", "expense",
	    IF(transactions[transaction_type_unsorted] = "Příjem", "income",
	    "N/A"
	    )))))
	    )
    )

Creating a Date table:

    dates = 
    ADDCOLUMNS(
	    CALENDARAUTO(),
	    "year", YEAR([Date]),
	    "year_quarter", YEAR([Date]) & "_Q" & QUARTER([Date]),
	    "year_month", FORMAT([Date], "YYYY_MM"),
	    "quarter", "Q" & QUARTER([Date]),
	    "month", FORMAT([Date], "MMMM"),
	    "month_short", FORMAT([Date], "MMM"),
	    "month_number", MONTH([Date]),
	    "week_number", WEEKNUM([Date]),
	    "weekday", FORMAT([Date], "dddd"),
	    "weekday_short", FORMAT([Date], "ddd"),
	    "week_start", [Date] - WEEKDAY([Date], 2) + 1,
	    "week_end", [Date] + (7 - WEEKDAY([Date], 2)),
	    "week_of_month", INT((DAY([Date]) - 1) / 7) + 1,
	    "is_weekday", IF(WEEKDAY([Date], 2) <= 5, TRUE, FALSE),
	    "day", DAY([Date]),
	    "day_of_year", DATEDIFF(DATE(YEAR([Date]), 1, 1), [Date], DAY) + 1
    )

## Some DAX Formulas Used in Measures

**1. Expenses**

    abs_expenses_in_DKK = ABS([expenses_in_DKK_negative_value])
	--Expenses in absolute value:


    abs_expenses_in_DKK MoM% = 
	-- Calculates the Month-over-Month (MoM) percentage change in absolute expenses (in DKK, transactions converted to DKK using historical exchange rates)  
	-- Compares the current month’s absolute expenses with the previous month’s 
	
	VAR __PREV_MONTH = 
	    CALCULATE(
	        [abs_expenses_in_DKK], -- Retrieves the total absolute expenses for the previous month 
	        DATEADD('dates'[Date], -1, MONTH) -- Shifts the date context back by 1 month
	    )
	
	RETURN
		DIVIDE(
	        [abs_expenses_in_DKK] - __PREV_MONTH, -- Difference between current and previous month
	        __PREV_MONTH -- Divides by the previous month's expenses to get percentage change
	    )


    avg_monthly_expense_by_selected_years = 
	-- Calculates the average monthly expense (in DKK, transactions converted to DKK using historical exchange rates) for the selected years  
	-- The result changes dynamically based on the slicer selection (e.g., selected year)  
	
	CALCULATE(  
	    AVERAGEX(  
	        SUMMARIZE(  
	            transactions,  
	            dates[year], -- Groups by year  
	            dates[month], -- Groups by month  
	            "Expenses",  
	            SUMX(  
	                FILTER(transactions, transactions[transaction_type] = "expense"), -- Filters for expense transactions 
	                ABS(transactions[transaction_DKK]) -- Uses absolute values of the transaction amounts in DKK  
	            )  
	        ),  
	        [Expenses] -- Calculates the average of expenses for each group  
	    ),  
	    ALLSELECTED(transactions) -- Applies all selected filters (like the slicer) to the transactions table  
	)


    avg_spent_MTD = 
	-- Calculates the average amount of expense transactions in absolute values  
	-- Uses TOTALMTD to limit the calculation to the current month (based on date context)  
	-- USE CASE = fx. in a card visual to show the average expense or in tooltip, e.g., for a specific year (when filtered by year)
	
	TOTALMTD(
	    AVERAGEX(
	        FILTER( -- Filters for expense transactions only 
	            transactions, 
	            transactions[transaction_type] = "expense"
	        ), 
	        ABS(transactions[transaction_DKK]) -- Uses absolute values of the transaction amounts in DKK  
	    ),
	    dates[Date] -- Applies month-to-date logic based on the date table
	)


    expenses_cumulative_prevY = 
    	-- Shows cumulative expenses for previous year
	-- USE CASE - eg. line chart comparing current and previous year expenses
	
	VAR _maxYear = CALCULATE(MAX(dates[year]), ALL(dates)) -- Get overall latest year
	VAR _previousYear = _maxYear - 1
	VAR _currentDateInAxis = MAX(dates[Date]) -- Get the date from the visual axis context
	
	-- Calculate the corresponding date in the previous year to set the upper limit
	VAR _endDateForPreviousYearCalc = DATE( _previousYear, MONTH(_currentDateInAxis), DAY(_currentDateInAxis) )
	
	RETURN
	CALCULATE(
	    [abs_expenses_in_DKK],
	    FILTER(
	        ALL(dates), -- Use ALL to clear incoming filters
	        dates[year] = _previousYear &&
	        dates[Date] <= _endDateForPreviousYearCalc -- Sum up to the equivalent date in the previous year
	    )
	)


    expenses_DKK_negative_value = 
	-- Calculates the total expenses in DKK (transactions converted to DKK using historical exchange rates) as a NEGATIVE value  
	-- Used for fx. visualizations comparing income and expenses  
	
	SUMX(  
	    FILTER(  
	        transactions,  
	        transactions[transaction_type] = "expense" -- Filters only expense transactions  
	    ),  
	    transactions[transaction_DKK]
	)


    expenses_DKK_in_negative_value_MoM% = 
	-- Calculates the Month-over-Month (MoM) percentage change in expenses (in DKK, transactions converted to DKK using historical exchange rates)  
	-- Ensures expenses remain as negative values for proper trend analysis  
	
	IF(  
	    ISFILTERED(transactions[date]), -- Ensures the measure is used with the correct date hierarchy  
	    ERROR("Time intelligence quick measures can only be grouped or filtered by the Power BI-provided date hierarchy or primary date column."), -- Returns an error if an incorrect date field is used  
	
	    VAR __PREV_MONTH =  
	        CALCULATE(  
	            [expenses_DKK_negative_value], -- Retrieves total expenses (as negative values) for the previous month  
	            DATEADD(dates[Date], -1, MONTH) -- Shifts the date context back by 1 month  
	        )  
	
	    RETURN  
	        DIVIDE(  
	            [expenses_DKK_negative_value] - __PREV_MONTH, -- Difference between current and previous month expenses  
	            __PREV_MONTH, -- Divides by the previous month's expenses to get percentage change  
	            BLANK() -- Returns BLANK() instead of an error if __PREV_MONTH is zero  
	        )  
	)

**2. Income, Balance, Investments**

    income_in_DKK = 
	-- Calculates the total income (Příjem) in DKK (transactions converted to DKK using historical exchange rates) 
	-- Used for fx. visualizations comparing income and expenses  
	
	SUMX(  
	    FILTER(  
	        transactions,  
	        transactions[transaction_type] = "income" -- Filters only income transactions  
	    ),  
	    transactions[transaction_DKK] -- Sums the transaction amounts in DKK  
	)


    balance_non-negative = 
	-- Shows total balance in original currency, adjusted so only emerald_credit can be negative.
	-- Handles fx. negative balances in investment accounts where investments have been sold with gain -> new balance is therefore 0
	-- USE CASE = e.g. table showing account balances in original currency
	
	-- Calculate the original balance using the latest rates
	VAR OriginalBalance =
	    SUM(transactions[transaction])
	
	-- Get the Account name currently being evaluated in the visual context
	VAR CurrentAccount = 
	    SELECTEDVALUE(accounts[account])
	
	RETURN
	    -- Apply the logic: Allow kb_kreditka to be negative, others max out at 0
	    IF(
	        CurrentAccount = "emerald_credit",
	        OriginalBalance,          -- For emerald_credit, show the original calculated balance
	        MAX(0, OriginalBalance)   -- For all other accounts, show 0 if the balance is negative
	    )    
	

    latest_balance_DKK = 
    	-- Shows total balance (in DKK) (exchange rates as of the LATEST date from dataset)

	VAR LatestTransactionDate = MAX(transactions[date]) -- Get the latest date from the transactions table
	
	-- Define variables to store the latest exchange rates
	VAR LatestCZKRate = LOOKUPVALUE(exchange_rates[CZK], exchange_rates[Date], LatestTransactionDate)
	VAR LatestDKKRate = LOOKUPVALUE(exchange_rates[DKK], exchange_rates[Date], LatestTransactionDate)
	VAR LatestUSDRate = LOOKUPVALUE(exchange_rates[USD], exchange_rates[Date], LatestTransactionDate)
	VAR LatestGBPRate = LOOKUPVALUE(exchange_rates[GBP], exchange_rates[Date], LatestTransactionDate)
	VAR LatestBTCRate = LOOKUPVALUE(exchange_rates[BTC], exchange_rates[Date], LatestTransactionDate)
	VAR LatestETHRate = LOOKUPVALUE(exchange_rates[ETH], exchange_rates[Date], LatestTransactionDate)
	
	RETURN
	    SUMX( -- Iterates through each row of the transactions table
	        transactions,
	        SWITCH( -- Converts each particular currency ID (fx. ID 2 is CZK) into DKK
	            transactions[currency_id],
	            2, transactions[transaction] / LatestCZKRate * LatestDKKRate,
	            3, transactions[transaction] / LatestUSDRate * LatestDKKRate,
	            4, transactions[transaction] * LatestDKKRate,
	            5, transactions[transaction] / LatestETHRate * LatestDKKRate,
	            6, transactions[transaction] / LatestBTCRate * LatestDKKRate,
	            7, transactions[transaction] / LatestGBPRate * LatestDKKRate,
	            transactions[transaction]
	        )
	    )


    transactions_DKK_running_total = 
	-- Calculates the running total of transactions in DKK over time (transactions converted to DKK using historical exchange rates)
	-- Uses ALLSELECTED to respect any filters applied to the dates table
	-- Can be used in area/line chart for visualization of balance over time
	
	CALCULATE(  
	    SUM(transactions[transaction_DKK]), -- Summing up all transactions converted to DKK  
	
	    FILTER(  
	        ALLSELECTED(dates[Date]), -- Ensures the calculation considers only the selected date range  
	        ISONORAFTER( -- Includes all dates from the latest selected date backward 
	            dates[Date], 
	            MAX(dates[Date]), 
	            DESC
	        )  
	    )  
	)    


    latest_invested_DKK = 
    	-- Shows how much money (in DKK) was invested (exchange rates as of the LATEST date from dataset)
	-- Filters only NON-NEGATIVE transactions of INVESTMENTS account type (= no sold positions are taken into account) ->
	-- -> converts each transaction into DKK and then makes SUM
	
	VAR LatestTransactionDate = MAX(transactions[date]) -- Get the latest date from the transactions table
	
	-- Define variables to store the latest exchange rates
	VAR LatestCZKRate = LOOKUPVALUE(exchange_rates[CZK], exchange_rates[Date], LatestTransactionDate)
	VAR LatestDKKRate = LOOKUPVALUE(exchange_rates[DKK], exchange_rates[Date], LatestTransactionDate)
	VAR LatestUSDRate = LOOKUPVALUE(exchange_rates[USD], exchange_rates[Date], LatestTransactionDate)
	VAR LatestGBPRate = LOOKUPVALUE(exchange_rates[GBP], exchange_rates[Date], LatestTransactionDate)
	VAR LatestBTCRate = LOOKUPVALUE(exchange_rates[BTC], exchange_rates[Date], LatestTransactionDate)
	VAR LatestETHRate = LOOKUPVALUE(exchange_rates[ETH], exchange_rates[Date], LatestTransactionDate)
	
	RETURN
	    CALCULATE(
	        SUMX( -- Iterates through each row of the transactions table
	            transactions,
	            SWITCH( -- Converts each particular currency ID (fx. 2 is CZK) into DKK
	                transactions[currency_id],
	                2, transactions[transaction] / LatestCZKRate * LatestDKKRate,
	                3, transactions[transaction] / LatestUSDRate * LatestDKKRate,
	                4, transactions[transaction] * LatestDKKRate,
	                5, transactions[transaction] / LatestETHRate * LatestDKKRate,
	                6, transactions[transaction] / LatestBTCRate * LatestDKKRate,
	                7, transactions[transaction] / LatestGBPRate * LatestDKKRate,
	                transactions[transaction]
	            )
	        ),
	        accounts[account_type] = "investment", -- filters only investment accounts
	        transactions[transaction] >= 0 -- negative values are NOT taken into calculation = therefore no sold positions are taken into account
	    )

**3. Other**

    exchange_rates_date_txt = 
	-- Creates a text string displaying the latest exchange rate date used  
	-- Displays the date in a readable format (e.g., "Latest Exchange Rate Used (As of March 15, 2025)")  
	
	"Exchange Rate As Of " &  
	    FORMAT(MAX(transactions[date]), "MMMM DD, YYYY") -- Formats the latest transaction date as "Month Day, Year"


    last_year_month = 
    	-- Shows the last month in "year_month" format (eg. 2025_03) in transaction table
	-- USE CASE = use it in visual filter (year_motnh = TOP 1 of last_year_month) of Gains and Expenses charts, to see only last month
	
	CALCULATE(
	    SELECTEDVALUE(dates[year_month]), -- selects last month in year_month format
	    dates[Date] = -- filters dates table, so it only contains values of transaction's last month
	        EOMONTH( -- show End-of-month date of appropriate month
	            MAX(transactions[date]), -- selects last date
	            0 -- this month
	        )
	)


    account_currency = 
	-- Measure to determine and display the currency for each account.
	-- USE CASE = e.g. table showing account balances in original currency together with "balance_non-negative" measure
	
	-- Get the Account name currently being evaluated in the visual context.
	VAR CurrentAccount =
	    SELECTEDVALUE(accounts[account])
	
	-- Filter the transactions table to include only transactions for the current account.
	VAR AccountTransactions =
	    FILTER(
	        transactions,
	        transactions[account_id] IN (
	            SELECTCOLUMNS(
	                FILTER(
	                    accounts,
	                    accounts[account] = CurrentAccount -- Filter accounts table for the current account.
	                ),
	                accounts[account_id] -- Select the account ID of the current account.
	            )
	        )
	    )
	
	-- Get the first non-blank currency ID from the transactions of the current account.
	VAR FirstCurrencyId =
	    FIRSTNONBLANK(
	        SELECTCOLUMNS( -- Create a table with only the currency IDs from the filtered transactions.
	            AccountTransactions, 
	            transactions[currency_id]
	        ), 
	        1 -- Placeholder value (not really used here).
	    )
	
	-- Lookup the currency name from the 'currencies' table based on the retrieved currency ID.
	VAR CurrencyName =
	    LOOKUPVALUE(
	        currencies[currency], -- The column in the 'currencies' table containing the currency name.
	        currencies[currency_id], -- The column in the 'currencies' table containing the currency ID.
	        FirstCurrencyId -- The currency ID obtained from the transactions.
	    )
	RETURN
	    CurrencyName -- Return the currency name.

## Contributing

Contributions to this project are welcome! Feel free to fork the repository, make improvements, and submit pull requests.

## License

This project is licensed under the [MIT License](LICENSE).

## Authors

- [Jan Dokoupil](https://github.com/jdok8)
