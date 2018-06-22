# degiro-java-client

Unofficial DeGiro stock boker java API client.

This library implements all DeGiro primitive operations. It provides the same functionality of DeGiro web and makes it easier to automate your portfolio management. DeGiro java client provides a set of methods and objects that allow you to perform the same interactions as with the web trader. DeGiro could change their API in any moment. 

If you have any questions, please open an issue.

## Usage

### Obtain a DeGiro instance
Add {maven_publish_pending} artifact to your project and then use ```DeGiroFactory``` to obtain a ```DeGiro``` instance.

```java
DCredentials creds = new DCredentials() {

        @Override
        public String getUsername() {
           return "YOUR_USERNAME";
        }

        @Override
        public String getPassword() {
            return "YOUR_PASSWORD";
        }
    };

DeGiro degiro = DeGiroFactory.newInstance(creds);
```
If you don't want to create a new DeGiro session on each execution instantiate a DeGiro object with a DPersistentSession. DeGiro API will try to reuse session values (if previous session is expired a new one is obtained and stored).

```java
DeGiro degiro = DeGiroFactory.newInstance(creds, new DPersistentSession("/path/to/session.json"));
```

### Get account data

```java
//Obtain current orders
List<DOrder> orders = degiro.getOrders();

//Obtain current portfolio
DPortfolioProducts portfolioProducts = degiro.getPortfolio();

//Obtain portfolioSummary
DPortfolioSummary portfolioSummary = degiro.getPortfolioSummary();

// Get cash funds
DCashFunds cashFunds = degiro.getCashFunds();

// Get last executed transactions
DLastTransactions lastTransactions = degiro.getLastTransactions();

// Get transactions between dates 
Calendar c = Calendar.getInstance();
Calendar c2 = Calendar.getInstance();
c.add(Calendar.MONTH, -1);
DTransactions transactions = degiro.getTransactions(c, c2);
```
### Search products

```java

// Search products by text, signature:
// DProductSearch searchProducts(String text, DProductType type, int limit, int offset);
DProductSearch ps = degiro.searchProducts("telepizza", DProductType.ALL, 10, 0);
for (DProductDescription product : ps.getProducts()) {
    System.out.println(product.getId() + " " + product.getName());
}

// Get product info by id, signature:
// DProducts getProducts(List<Long> productIds);
List<Long> productIds = new ArrayList<>();
productIds.add(1482366L); // productId obtained in (orders, portfolio, transactions, searchProducts....)
degiro.getProducts(productIds);
DProductDescriptions products = degiro.getProducts(productIds);

for (DProductDescription value : products.getData().values()) {
    System.out.println(value.getId() + " " + value.getName());
}

```

### Subscribe to product price updates

```java

// Register a price update listener (called on price update)
degiro.setPriceListener(new DPriceListener() {
    @Override
    public void priceChanged(DPrice price) {
        System.out.println(new GsonBuilder().setPrettyPrinting().create().toJson(price));
    }
});

// Create a vwdIssueId list. Note that vwdIssueId is NOT a productId (vwdIssueId is a DProduct field).
List<Long> vwdIssueIds = new ArrayList<>(1);
vwdIssueIds.add(280099308L); // Example product vwdIssueId
degiro.subscribeToPrice(vwdIssueIds); // Callable multiple times with different products. 

// You need some type of control loop, background thread, etc... to prevent JVM termination (out of this scope)
while (true) {
   Thread.sleep(1000);
}

```
By default, price updates are checked every 15 seconds. Polling frequency can be changed:

```java
degiro.setPricePollingInterval(1, TimeUnit.MINUTES);
```
Clear all subscriptions:

```java
degiro.clearPriceSubscriptions();
```

### Order management
Orders are placed in two steps: check order (to ensure order factibility) and confirmation. When DConfirmation status is 0 then the order is placed successfully.


```java
// Generate a new order. Signature:
// public DNewOrder(DOrderAction action, DOrderType orderType, DOrderTime timeType, long productId, long size, BigDecimal limitPrice, BigDecimal stopPrice)

DNewOrder order = new DNewOrder(DOrderAction.SELL, DOrderType.LIMITED, DOrderTime.DAY, 1482366, 20, new BigDecimal("4.5"), null);

DOrderConfirmation confirmation = degiro.checkOrder(order);

if (!Strings.isNullOrEmpty(confirmation.getConfirmationId())) {
    DPlacedOrder placed = degiro.confirmOrder(order, confirmation.getConfirmationId());
    if (place.getStatusId() != 0) {
        throw new RuntimeException("Order not placed: " + place.getStatusText());
    }
}
```
Order update example:

```java
// Update an order. Signature:
// DPlacedOrder updateOrder(DOrder order, BigDecimal limit, BigDecimal stop);
DPlacedOrder updated = degiro.updateOrder(order, new BigDecimal("0.04"), null); // obtained in getOrders()
if (updated.getStatusId() != 0) {
    throw new RuntimeException("Order not updated: " + updated.getStatusText());
}
```


Order delete example:

```java
DPlacedOrder deleted = degiro.deleteOrder(orderId); // orderId obtained in getOrders() 
if (deleted.getStatusId() != 0) {
    throw new RuntimeException("Order not deleted: " + deleted.getStatusText());
}
```
