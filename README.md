Sylius Loyalty Bundle
==============

Points
------

Points should behave like a currency: there should be an exchange rate for points to the default currency of the website. 
We need this behavior because users should be able to use points to lower the amount of their cart total upon completion.

Points should also have state in order to avoid fraud. Imagine this scenario: John Doe is placing an order of 1000€ on the website, getting back 20 points that worth 20€ for his order. After he places the order he calls customer support to cancel the order.

If points don’t have state, the order gets cancelled and the user remains with 20€ in his account that he can use to place an order without paying at all for it.

Points state should be handled via a machine state in relation with the order status:

 - if the order gets delivered - the points should become available
 - if the order gets closed (delivered + 14 days for returns) - the points
   should become available - this is available for those using returns
   in their order workflow

Points could be a virtual currency or a real currency, depending on the configuration.

Levels
------

Points should be awarded based on a matrix of levels. Let’s say we have two levels:

 - basic
 - premium

The admin should be able to create how many levels he want. Example:

 - basic
 - premium
 - vip

Moving from a level to another should be done based on a strategy. Potential strategies could be:

 - total customer lifetime value
 - total no. of orders
 - no. of days since registration

Levels should have values of how many points are awarded when a user reaches a certain level.

Levels should have types:

 - simple
 - interval

Simple levels will have one value: if you are a “premium” customer you get 5% cashback on each order.

Interval level will have a value for each interval: if you are a “premium” customer you get 5% cashback for orders below 100€ and 10% for orders below 100€.

```gherkin
  Scenario Template: MyAccount - BestValue Club Cashback algorithm
    Given I am on "<level>" club level
    And I have "<life_time_value>" total lifetime value
    When I make a "<amount>" acquisition
    Then I get "<cashback>" cashback
    And I am "<new_level>" club level
    Examples:
          | life_time_value | level          | amount  | cashback  | new_level |
      | 0€              | basic =< 100   | 100€    | 5€        | basic     |
      | 100€            | basic > 100    | 200€    | 10€       | basic     |
      | 1500€           | premium =< 100 | 100€    | 5€        | premium   |
      | 1900€           | premium > 100  | 200€    | 20€       | vip       |
      | 2200€           | vip =< 100     | 100€    | 15€       | vip       |
      | 4000€           | vip > 100      | 500€    | 75€       | vip       |
```

Award mechanism
---------------

Based on the initial requirements, the user should have received points for multiple actions: 

 - register
 - login
 - time since registration
 - order placement
 
We should go first on order placement as this brings the more business value to the table. Upon finishing the order, there should be a hook mechanism in place that should calculate:

  - the current loyalty level of the current user
  - the kind of level
   (simple or interval)
  - based on the order amount the point amount
  - saving this as “pending” points linked to the current order

MyAccount
---------

   For this we have some Scenarios written. they’re not in their best
   form, but we can work on them.
   
```gherkin
  Scenario: Cashback points "pending" status
    Given the current date is "01.02.2016"
    And I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
    And I see "0 points available"
    And points move from "pending" to "available" if "current date > order date + 15 days"
    When I made an order of 200€
    And I use 0 points
    And order ID is "100"
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status                  |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Pending until 16.01.2016|
    And I see "22 points available"

  # we decided to award points after 15 days from order date
  # to make sure we avoid fraud
  # and keep complexity at a minimum level





  Scenario: Cashback points "used" status
    Given the current date is "01.02.2016"
    And I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
    And I see "30 points available"
    When I made an order of 200€
    And I use 8 points
    And order ID is "180"
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status          |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Used 5 for #180 |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Used 3 for #180 |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available       |
    And I see "22 points available"








  Scenario: Cashback points "cancelled" status
    Given I the current date is "01.02.2016"
    And I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
    And I see "30 points available"
    When I made an order of 200€
    And I use 0 points
    And order ID is "180"
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
      | 180      | 01.02.2016 | site    | 200€              | 0            | 200€             | 10             | 01.02.2017   | Available |
    But I cancelled the order with ID "180"
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
      | 180      | 01.02.2016 | site    | 200€              | 0            | 200€             | 10             | 01.02.2017   | Cancelled |
    And I see "22 points available"



  Scenario: Cashback points manually awarded
    Given I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
    And current date is "15.02.2016"
    When an admin user gives 10 points
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
      | n/a      | n/         | manual  | n/a               | n/a          | n/a              | 10             | 15.02.2017   | Available |



  Scenario: Cashback points "expired" status
    Given I the current date is "01.02.2016"
    And I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status    |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Available |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Available |
      | 140      | 03.01.2016 | site    | 300€              | 0            | 300€             | 15             | 03.01.2017   | Available |
    And I see "30 points available"
    When the date is "01.07.2017"
    But I did not make any orders between "01.02.2016" and "01.07.2017"
    Then I have points status:
      | Order ID | Order date | Channel | Total order value | Used points  | Paid order value | Points awarded | Availability | Status                |
      | 100      | 01.01.2016 | site    | 100€              | 0            | 100€             | 5              | 01.01.2017   | Expired on 01.01.2017 |
      | 120      | 02.01.2016 | site    | 200€              | 0            | 200€             | 10             | 02.01.2017   | Expired on 02.01.2017 |
```

Checkout integration
--------------------

Points should behave the same as vouchers in checkout. User should have an input field where they can insert the total amount of points they want to use for an order based on the amount available in their account.


Admin
-----

The admin should be able to adjust points and there should be audit log for that.
