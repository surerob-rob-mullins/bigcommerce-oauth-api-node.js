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
```javascript
// Load Dependencies:
var Connection = require('./connection');

// Initialize new API Connection:
var api = new Connection({
  hash:  'the store hash',
  token: 'the store oAuth API token',
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

//*** Execute the Run Sale program! ***//
runSale();

/**
 * Begin the sale/product update process.
 * You can optionally wrap this in a Promise if this is part of
 * a larger process.
 */
function runSale() {
  // Create the Clearence Category:
  createCategory('Clearence!').then(function(cid) {
    // Determine total # of product pages, with 250 products per page:
    getTotalProductsCount().then(function(totalProducts) {
      // Determine the Total Number of Pages, with a limit of 250 products per page:
      var totalPages = ceil(totalProducts / limit); // Total Pages = (Total Products / Products Per Page)     
      
      /** ---------------------------------
       * NOTE: This will retrieve all of the products, page by page, asychronously and in parallel!
       */
      // Get all products by page, 250 products per page.
      var updated = 0; // Counter to track when all products finished update process.
      for (x = 1; x <= totalPages; x++) {
        // Get a single page of products:
        getProducts(page, limit).then(function(products) {
          // Parse through each individual product...
          products.forEach(function(element, index) {
            // For Each product, Calculate the Sales Price and Construct the new Category List:
            var salesPrice = (products[i].price / 2);
            var categories = products[i].categories.push(cid); // We are pushing the 'Clearence' category ID to the existing group of category IDs for this product.
            updateProduct(products[i].id, {sale_price: salesPrice, categories: categories}).then(function(res) {
              updated++; // Increment updated products count.
              // Check if all products have been processed:
              if (updated === totalProducts) {
                console.log('Product Sale Update Process has Finished!');
                return 1; // End runSale() function. Process Complete!
              }
            }).catch(function(e) {
              updated++; // Increment updated products. Even if the product failed, we just want to track when all have finished processed.
              console.log('Caught error updating product.');
            });
          }); // END products.forEach
        }).catch(function(e) {
          console.log('Caught error getting a page of products.');
        });
      } // END for 
      //---------------------------------//
    
    }).catch(function(e) {
      console.log('Caught error getting total product count.');
    });
  }).catch(function(e) {
    console.log('Caught error creating category.');
  }); 
}


/**
 * Creates a new category with a given name.
 * @param name <string> - The name of the category to create.
 * @return Promise - Fulfilled with category ID on success. Reject on fail.
 */
function createCategory(name) {
  return new Promise(function(fulfill, reject) {
    // Reject if 'name' parameter not set:
    if (typeof name === 'undefined') {
      return reject('Error: Category name not provided.');
    }
    // Perform API POST - Create the Category:
    api.post('/categories', {name: name}).then(function(category) {
      fulfill(category.id);
    }).catch(function(e) {
      reject('Error: Could not create category. Info: ', e);
    });
  });
}

/**
 * Gets the total number of products for the store. 
 * @return Promise - Fulfilled with the total product count on success. Reject on fail.
 */
function getTotalProductsCount() {
  return new Promise(function(fulfill, reject) {
    // Perform API GET - Get Total Number of Products:
    api.get('/products/count').then(function(count) {
      fulfill(count.count); 
    }).catch(function(e) {
      reject('Error: Could not get products count. Info: ', e);
    });
  });
}

/**
 * Gets a page of products. 
 * @param page  <int> - The page number to retrieve.
 * @param limit <int> - The max number of products per page.
 * @return Promise    - Fulfilled with array of products on success. Reject on fail.
 */
function getProducts(page, limit) {
  return new Promise(function(fulfill, reject) {
    // Reject if 'name' || 'limit' parameter not set:
    if (typeof page === 'undefined') {
      return reject('Error: Page not provided.');
    } else if (typeof limit === 'undefined') {
      return reject('Error: Limit not provided.');
    }
    // Perform API GET - Get a single page of products:
    api.get('/products?limit=' +limit +'&page=' +page).then(function(products) {
      fulfill(products);
    }).catch(function(e) {
      reject('Error: Could not get page of products. Info: ', e);
    });
  });
}

/**
 * Updates an individual product by ID.
 * @param id <int>       - The ID of the product to update.
 * @param update <mixed> - Object containing the product update properties.
 * @return Promise       - Fulfilled with true on success. Reject on fail.
 */
function updateProduct(id, update) {
  return new Promise(function(fulfill, reject) {
    // Reject if 'name' || 'limit' parameter not set:
    if (typeof id === 'undefined') {
      return reject('Error: Product ID not provided.');
    } else if (typeof update === 'undefined') {
      return reject('Error: Product Update Object not provided.');
    }
    // Perform API PUT - Update the individual product:
    api.put('/products/' +.id, update).then(function(res) {
      console.log('Successfully updated product with ID=%d, sale price = %d', res.id, res.sale_price);
      fulfill(true);
    }).catch(function(e) {
      console.log("Unable to update product with ID = %d", +id);
      reject('Error: Could not update product. Info: ', e);
    });
  });
}


// And there you have it. The cool thing here is that the connection will automatically handle the rate-limiting for you. 
// Hope you enjoy and find this useful :) ~ rob ~
        
```
