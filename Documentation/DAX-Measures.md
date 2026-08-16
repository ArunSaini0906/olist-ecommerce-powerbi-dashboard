# DAX Measures

# Core KPIs

## Total Revenue

Total Revenue =
SUM('Order Items'[Revenue])

## Total Orders

Total Orders =
DISTINCTCOUNT('Orders'[order_id])

## Total Customers

Total Customers =
DISTINCTCOUNT('Customers'[customer_unique_id])

## Average Order Value

Average Order Value =
DIVIDE(
    [Total Revenue],
    [Total Orders]
)

## Total Freight

Total Freight = 
SUM('Order Items'[freight_value])

## Average Rating

Average Rating = 
AVERAGE('Reviews'[review_score])

# Growth Analysis

## Revenue Growth %

Revenue Growth % = 
VAR CurrentRevenue =
    [Total Revenue]

VAR PreviousYearRevenue =
    CALCULATE(
        [Total Revenue],
        DATEADD(
            'DimDate'[Date],
            -1,
            YEAR
        )
    )

RETURN
    IF(
        NOT ISBLANK(PreviousYearRevenue),
        DIVIDE(
            CurrentRevenue - PreviousYearRevenue,
            PreviousYearRevenue
        ),
        BLANK()
    )

## Order Growth %

Order Growth % = 
VAR CurrentOrders =
    [Total Orders]

VAR PreviousOrders =
    CALCULATE(
        [Total Orders],
        DATEADD(
            'DimDate'[Date],
            -1,
            YEAR
        )
    )

RETURN
DIVIDE(
    CurrentOrders - PreviousOrders,
    PreviousOrders
)

# Customer Analytics

## Repeat Customers

Repeat Customers = 
COUNTROWS(
    FILTER(
        VALUES(
            'Customers'[customer_unique_id]
        ),
        CALCULATE(
            DISTINCTCOUNT(
                'Orders'[order_id]
            )
        ) > 1
    )
)

## Repeat Customer Rate

Repeat Customer Rate = 
DIVIDE(
    [Repeat Customers],
    [Total Customers]
)

## Average Customer Spend

Average Customer Spend = 
DIVIDE(
    [Customer Revenue],
    [Total Customers]
)

## Orders per Customer

Orders per Customer = 
DIVIDE(
    [Total Orders],
    [Total Customers]
)

# Product & Category

## Revenue Contribution %

Revenue Contribution % = 
DIVIDE(
    [Total Revenue],
    CALCULATE(
        [Total Revenue],
        REMOVEFILTERS('Products'[Category])
    )
)

## Average Revenue per Category

Average Revenue / Category = 
DIVIDE(
    [Total Revenue],
    DISTINCTCOUNT('Products'[Category])
)

# Operations

## Average Delivery Days

Average Delivery Days = 
AVERAGEX(
    FILTER(
        'Orders',
        'Orders'[order_status] = "delivered"
            && NOT ISBLANK(
                'Orders'[order_delivered_customer_date]
            )
            && NOT ISBLANK(
                'Orders'[order_purchase_timestamp]
            )
    ),
    DATEDIFF(
        'Orders'[order_purchase_timestamp],
        'Orders'[order_delivered_customer_date],
        DAY
    )
)

## On-Time Delivery %

On-Time Delivery % = 
VAR DeliveredOrders =
    FILTER(
        'Orders',
        'Orders'[order_status] = "delivered"
            && NOT ISBLANK(
                'Orders'[order_delivered_customer_date]
            )
            && NOT ISBLANK(
                'Orders'[order_estimated_delivery_date]
            )
            && 'Orders'[order_delivered_customer_date]
                <= 'Orders'[order_estimated_delivery_date]
    )

VAR TotalDelivered =
    CALCULATE(
        [Total Orders],
        'Orders'[order_status] = "delivered"
    )

RETURN
DIVIDE(
    COUNTROWS(DeliveredOrders),
    TotalDelivered
)

## Late Delivery %

Late Delivery % = 
1 - [On-Time Delivery %]

## Average Delivery Delay Days

Average Delivery Delay Days = 
VAR LateOrders =
    FILTER(
        'Orders',
        NOT ISBLANK('Orders'[order_delivered_customer_date])
            &&
        NOT ISBLANK('Orders'[order_estimated_delivery_date])
            &&
        'Orders'[order_delivered_customer_date]
            >
        'Orders'[order_estimated_delivery_date]
    )

RETURN
    AVERAGEX(
        LateOrders,
        DATEDIFF(
            'Orders'[order_estimated_delivery_date],
            'Orders'[order_delivered_customer_date],
            DAY
        )
    )

# Executive Insights

## Customer by Segment

Customers by Segment = 
VAR SelectedSegment =
    SELECTEDVALUE('Customer Segment'[Segment])

VAR CustomerRevenueTable =
    FILTER(
        ADDCOLUMNS(
            ALLSELECTED('Customers'[customer_unique_id]),
            "@CustomerRevenue",
                CALCULATE([Total Revenue])
        ),
        NOT ISBLANK([@CustomerRevenue])
    )

VAR CustomerCount =
    COUNTROWS(CustomerRevenueTable)

VAR P80Revenue =
    PERCENTILEX.INC(
        CustomerRevenueTable,
        [@CustomerRevenue],
        0.80
    )

VAR P50Revenue =
    PERCENTILEX.INC(
        CustomerRevenueTable,
        [@CustomerRevenue],
        0.50
    )

RETURN
    SWITCH(
        SelectedSegment,

        "High Value",
            COUNTROWS(
                FILTER(
                    CustomerRevenueTable,
                    [@CustomerRevenue] >= P80Revenue
                )
            ),

        "Medium Value",
            COUNTROWS(
                FILTER(
                    CustomerRevenueTable,
                    [@CustomerRevenue] >= P50Revenue
                        && [@CustomerRevenue] < P80Revenue
                )
            ),

        "Low Value",
            COUNTROWS(
                FILTER(
                    CustomerRevenueTable,
                    [@CustomerRevenue] < P50Revenue
                )
            ),

        CustomerCount
    )

## Customer Segment %   

Customer Segment % = 
DIVIDE(
    [Total Customers],
    CALCULATE(
        [Total Customers],
        ALL('Customer Segment'[Segment])
    )
)

