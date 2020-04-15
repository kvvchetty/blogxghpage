Mongo-DB : A data model scenario

A proto type document

Person

![](./media/image1.png){width="2.9270833333333335in"
height="1.7604166666666667in"}

*MyCompany has just launched their brand new OneDroid phone range to the
eager reception of the consumer market. Your task is to build a
e-commerce system to take advantage of this huge opportunity and the
stock we got allocated.*

**The Product Catalog**
-----------------------

The first step is to design the schema for the website. Consider an
initial product schema:

{

sku: \"111445GB3\",

title: \"Simsong One mobile phone\",

description: \"The greatest Onedroid phone on the market \.....\",

manufacture\_details: {

model\_number: \"A123X\",

release\_date: new ISODate(\"2012-05-17T08:14:15.656Z\")

},

shipping\_details: {

weight: 350,

width: 10,

height: 10,

depth: 1

},

quantity: 99,

pricing: {

price: 1000

}

}

This data model stores physical details like manufacturing and shipping
information as embedded documents in the larger product document, which
makes sense because these physical details are unique features of the
product. This gives the document \"strong data locality,\" which allows
easy mapping in an object oriented environment.

To insert the document in the products collection, use the following
commands.

mongo

use ecommerce

db.products.insert({

sku: \"111445GB3\",

title: \"Simsong One mobile phone\",

description: \"The greatest Onedroid phone on the market \.....\",

manufacture\_details: {

model\_number: \"A123X\",

release\_date: new ISODate(\"2012-05-17T08:14:15.656Z\")

},

shipping\_details: {

weight: 350,

width: 10,

height: 10,

depth: 1

},

quantity: 99,

pricing: {

price: 1000

}

})

The first command (mongo) starts the mongodb console and connects to the
local Mongo DB console on localhost and port 27017. The next chooses the
ecommerce database (use ecommerce) and the third inserts the product
document in the products collection. Going forward all commands are
assuming you are in the Mongo DB shell using the ecommerce database.

The products data model has a unique sku that identifies the product,
title, description, a stock quantity, and pricing information about the
item.

All products have categories. In the case of the *Simsong One* it\'s a
15G phone and also has a FM receiver. As a result, This product falls
into both the mobile/15G and the radio/fm categories. Add the categories
to the existing document, with the following update() operation:

db.products.update({sku: \"111445GB3\"}, {\$set: { categories:
\[\'mobile/15G\', \'mobile/fm\'\] }});

To support efficient queries using the categories field, add an index on
the categories field for the products collection:

db.products.ensureIndex({categories:1 })

This returns all the products for a specific category using the index
and an anchored regular expression. As long as the regular expression is
case sensitive and anchored, MongoDB will use the index to return the
query. For example, fetch all the products in the category that begins
with mobile/fm:

db.products.find({categories: /\^mobile\\/fm/})

To be able to provide a list of all the products in a category, amend
the data model with a collection of documents for each category. In this
collection, each document represents a category and contains the path
for that category in category tree. These documents would resemble the
following.

{

title: \"Mobiles containing a FM radio\",

parent: \"mobile\",

path: \"mobile/fm\"

}

Insert the document into the categories collection and add indexes to
this collection:

db.categories.insert({title: \"Mobiles containing a FM radio\", parent:
\"mobile\", path: \"mobile/fm\"})

db.categories.insert({title: \"Mobiles with 15G support\", parent:
\"mobile\", path: \"mobile/15G\"})

db.categories.ensureIndex({parent: 1, path: 1})

db.categories.ensureIndex({path: 1})

There are two paths in each category: this allows the application to use
the same method to find all categories for a specific category root as
used for finding products by category. For example, to return all
sub-categories of the category \"mobile\", use the following query:

db.categories.find({parent: /\^mobile/}, {\_id: 0, path: 1})

This will return the following documents:

{\"path\": \"mobile/fm\"}

{\"path\": \"mobile/15G\"}

Using these path values, the application can use this method to access
the category tree and extract more sub-categories with a single index
supported query. Furthermore, the application can pull all the documents
for a specific category using this path value.

### The Cart

A cart in an e-commerce system, allows users to reserve items from the
inventory and keep them until they check out and pay for the items. The
application must ensure that at any point in time there are not more
items in carts than there are in stock *and* that if the users abandons
the cart, the application must return the items from the cart to the
inventory without loosing track of any objects. Take the following
document, which models the cart:

{

\_id: \"the\_users\_session\_id\",

status:\'active\'

quantity: 2,

total: 2000,

products: \[\]

}

The products array contains the list of products the customer intends to
purchase. Use the following insert() operation to create the cart:

db.carts.insert({

\_id: \"the\_users\_session\_id\",

status:\'active\',

quantity: 2,

total: 2000,

products: \[\]});

If the inventory had 99 items, after this operation, the inventory
should have 97 items. To prevent \"overselling,\" the application must
move items from the inventory to the cart. To support these operations
applications must perform a set of updates, and be able \"rollback\"
changes if something goes awry. Begin by adding a product to the
customer\'s cart with the following operation:

db.carts.update({

\_id: \"the\_users\_session\_id\", status:\'active\'

}, {

\$set: { modified\_on: ISODate() },

\$push: {

products: {

sku: \"111445GB3\", quantity: 1, title: \"Simsong One mobile phone\",
price:1000

}

}

});

Then, check to ensure that the inventory can support adding the product
to the customers cart:

db.products.update({

sku: \"111445GB3\", quantity: {\$gte: 1}

}, {

\$inc: {quantity: -1},

\$push: {

in\_carts: {

quantity:1, id: \"the\_users\_session\_id\", timestamp: new ISODate()

}

}

})

This operation only succeeds if there is sufficient inventory, and the
application must detect the operation\'s success or failure. Call
getLastError to fetch the result of the attempted update:

if(!db.runCommand({getLastError:1}).updatedExisting) {

db.carts.update({

\_id: \"the\_users\_session\_id\"

}, {

\$pull: {products: {sku:\"111445GB3\"}}

})

}

If updatedExisting is false in the resulting document, the operation
failed and the application must \"roll back\" the attempt to add the
product to the users cart. This pattern ensures that the application
cannot have more products in carts than the available inventory.

In addition to simply adding objects to carts, there are a number of
cart related operations that the application must be able to support:

-   users may add or remove objects from the cart.

    > users may abandon a cart and the application must return items in
    > the cart to inventory.

The next sequence of operations allow the application to ensure that
carts are up to date and that the application has enough inventory to
cover it. Update the cart with the new quantity, using the following
update() operation:

var new\_quantity = 2;

var old\_quantity = 1;

var quantity\_delta = new\_quantity - old\_quantity;

db.carts.update({

\_id: \"the\_users\_session\_id\", \"products.sku\": \"111445GB3\",
status: \"active\"

}, {

\$set: {

modified\_on: new ISODate(),

\"products.\$.qty\": new\_quantity

}

})

Notice that there are some defined variables called new*quantity,
old*quantity and quantity\_delta to contain the new and previous
quantity in the cart as well as the delta that needs to be requested
from the inventory.

Now, remove the additional item from the inventory update the number of
items in the shopping cart:

db.products.update({

sku: \"111445GB3\",

\"in\_carts.id\": \"the\_users\_session\_id\",

quantity: {

\$gte: 1

}

}, {

\$inc: { quantity: (-1)\*quantity\_delta },

\$set: {

\"in\_carts.\$.quantity\": new\_quantity, timestamp: new ISODate()

}

})

Ensure the application has enough inventory for the operation. If there
is not sufficient inventory, the application must rollback the last
operation. The following operation checks for errors using getLastError
and rolls back the operation if it returns an error:

if(!db.runCommand({getLastError:1}).updatedExisting) {

db.carts.update({

\_id: \"the\_users\_session\_id\", \"products.sku\": \"111445GB3\"

}, {

\$set : { \"in\_carts.\$.quantity\": old\_quantity}

})

}

If a user abandons the purchase process or the shopping cart grows stale
and times out, the application must return cart content to the
inventory. This operation requires a loop that finds all expired or
canceled carts and then returns the content of each cart to the
inventory. Begin by finding all sufficiently \"stale\" carts, and use an
operation that resembles the following:

var carts = db.carts.find({status:\"expiring\"})

for(var i = 0; i \< carts.length; i++) {

var cart = carts\[i\]

for(var j = 0; j \< cart.products.length; j++) {

var product = cart.products\[i\]

db.products.update({

sku: product.sku,

\"in\_carts.id\": cart.\_id,

\"in\_carts.quantity\": product.quantity

}, {

\$inc: {quantity: item.quantity},

\$pull: {in\_carts: {id: cart.\_id}}

})

}

db.carts.update({

\_id: cart.\_id,

\$set: {status: \'expired\'}

})

}

This operation walks all products in each cart and returns them to the
inventory and removes cart identifiers from the in\_carts array in the
product documents. Once the application has returned all of the items to
the inventory, the application sets the cart\'s status to expired.

### Checkout

When the user clicks the \"confirm\" button in the checkout portion of
the application, the application creates an \"order\" document that
reflects the entire order. Consider the following operation:

db.orders.insert({

created\_on: new ISODate(\"2012-05-17T08:14:15.656Z\"),

shipping: {

customer: \"Peter P Peterson\",

address: \"Longroad 1343\",

city: \"Peterburg\",

region: \"\",

state: \"PE\",

country: \"Peteonia\",

delivery\_notes: \"Leave at the gate\",

tracking: {

company: \"ups\",

tracking\_number: \"22122X211SD\",

status: \"ontruck\",

estimated\_delivery: new ISODate(\"2012-05-17T08:14:15.656Z\")

},

},

payment: {

method: \"visa\",

transaction\_id: \"2312213312XXXTD\"

}

products: {

{quantity: 2, sku:\"111445GB3\", title: \"Simsong mobile phone\",
unit\_cost:1000, currency:\"USDA\"}

}

})

For a relational databases you might need to model this as a set of
tables: for orders, shipping, tracking, and payment. Using MongoDB one
can create a single document that is self-contained, easy to understand,
and simply maps into an object oriented application. After inserting the
this document the application must ensure inventory is up to date before
completing the checkout. Begin by setting the cart as finished, with the
following operation:

db.carts.update({

\_id: \"the\_users\_session\_id\"

}, {

\$set: {status:\"complete\"}

});

Use the following operation to remove the cart identifer from all
product records:

db.products.update({

\"in\_carts.id\": \"the\_users\_session\_id\"

}, {

\$pull: {in\_carts: {id: \"the\_users\_session\_id\"}}

}, false, true);

By using \"multi-update,\" which is the last argument in the update()
method, this operation will update all matching documents in one set of
operations.

The rich document capabilities atomic operation guarantees in MongoDB
makes it possible to model many different applications in MongoDB. Even
the rigorous requirements of conventional applications like e-commerce
system are possible in a document database. **This data model (i.e.
\"schema design,\")** is useful for developing applications around any
restricted resource system, not just e-commerce systems.

The close relationship match between object oriented application code
and documents leads to more simple data models and less glue code
between the data storage system and the application-level code.
