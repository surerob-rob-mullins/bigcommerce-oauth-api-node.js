# bigcommerce-oauth-api-node.js
A simple Promise based CRUD connection wrapper to interface with the BigCommerce oAuth API. Automatically re-tries requests that are rate-limited. 

#Requirements
This file requires the following node modules to work properly: <br/>
  1. promise <br/>
  2. request <br/>

You can install them as so: <br/>
`$ sudo npm install promise` <br/>
`$ sudo npm install request` 

#Features
1. Promise based. Every CRUD request returns a Promise for easy and organized Async control.
2. Rate-Limit Handling. For every request made, if that request is rate-limited, the app will sleep and automatically retry that request.
3. Connection Pooling. Can set a max number of connections, so that requests will be throttled if you attempt to exceed this number. 

#Usage
Usage is fairly simple. Just require the connection file into your app, and initialize it with the neccessary oAuth credentials. 
```
// Load Dependencies:
var Connection = require('./connection');

// Initialize new API Connection:
var api = new Connection({
  hash:  'the store hash'
  token: 'the store's oAuth API token',
  cid:   'your app client ID',
  host:  'https://api.bigcommerce.com' //The BigCommerce API Host
});

//**************** Basic Example Requests ****************//

//** Get the names of 10 products **//
api.get('/products?limit=10').then(function(products) {
  products.forEach(function(e, i) {
    console.log(products[i].name);
  });
}).catch(function(e) {
  // Catch any errors
  // NOTE: An error is thrown if BigCommerce returns a status other than 200 | 429
  console.log('Error on request - ' +e);
});

//----------------------------------//

//** Create a new product **//
// Define the minimum new product requirements. @see https://developer.bigcommerce.com/api/stores/v2/products#create-a-product
var newProduct = {
  name:  'Some new product',
  price: 19.99,
  type:  'physical',
  categories: [1, 2, 3],
  weight: 0.5
  availability: 'available'
};

// Make the POST request:
api.post('/products', newProduct).then(function(product) {
  console.log('Successfully created new product with ID of %d', response.id);
}).catch(function(e) {
  console.log('Error on request - ' +e);
});

//----------------------------------//

//** Update a product **//
// Update the price on product with ID 1. @see https://developer.bigcommerce.com/api/stores/v2/products#update-a-product
api.put('/products/1', {price: 29.99}).then(function(product) {
  console.log('Successfully updated product! - ' +product);
}).catch(function(e) {
  console.log('Error on request - ' +e);
});

//----------------------------------//

//**************** Advanced Examples (Not for the faint of heart) ****************//

/** 
 * Scenario: Business is having a store-wide clearance. Every single product is now %50 off, and 
 * every single product should be added into a new category called 'Half-Off' that we need to create,
 * in addition to the categories the product is currently assigned to. 
 *
 * Logic:
 *  1. Create the new category, and save its ID. 
 *  2. Determine the total number of products and API pages. 
 *  3. Get a collection of products, 250 at a time, and prepare its new price and category.
 *  4. Update the individual product. 
 *
 * The speed of this will be unmatched by any other language, due to the Async parallel nature of Node's HTTP requests. 
 * The speed for this will vary depending on the max number of connections you define. 
 */

// Create the new category - @see https://developer.bigcommerce.com/api/stores/v2/categories#create-a-category
api.post('/categories', {name: 'Half-Off!'}).then(function(category) {
  // New category created. 
  var category_id = category.id;
  // Now determine total number of products - @see https://developer.bigcommerce.com/api/stores/v2/products#get-a-product-count
  api.get('/products/count').then(function(count) {
    var totalProducts = count.count;
    // Prepare to get all products
    var limit = 250; // The max number of produts we can get per API request. 
    var totalPages = ceil(totalProducts / limit); // The total number of pages, at 250 products per page. 
    
    // Get all products:
    for (x=1; x<=totalPages; x++) {
      api.get('products?limit=' +limit +'&page=' +x).then(function(products) {
        // Parse through each product:
        products.forEach(function(e, i) {
          // For each product, determine the new price, and new category collection
          var newPrice = (products[i].price - (products[i].price * .5)); // 50% off
          var newCategories = products[i].categories.push(category_id); // Add the 'Half-Off' category ID to the product's existing category collection.
          // Finally, update the product:
          api.put('/products/' +products[i].id, {price: newPrice, categories: newCategories}).then(function(response) {
            console.log('Successfully updated product with ID=%d, new price = %d', response.id, response.price);
          }).catch(function(e) {
            console.log('Unable to update product - ' +e);
          });
        });
      }).catch(function(e) {
        console.log('Error getting products - ' +e);
      });
    }
  }).catch(function(e) {
    console.log('Error getting total product count - ' +e);
  });
}).catch(function(e) {
  console.log('Error creating the half-off category - ' +e);
});

// And there you have it. The cool thing here is that the connection will automatically handle the rate-limiting for you. 
// Hope you enjoy and find this useful :) ~ rob ~
        
```
