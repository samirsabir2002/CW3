<template>
  <link rel="stylesheet" href="/styles.css">
  <div>
    <header v-if="showProduct">
      <div class="title-bar">
  <h1>{{ sitename }}</h1>
  <div class="search-sort">
    <div class="search-container">
      <input type="text" v-model="searchQuery" placeholder="Search...">
    </div>
    <div class="sort-checkout-container">
      <button @click="toggleCheckout" :disabled="cart.length === 0" class="button">
  {{ cart.length }}
  <span class="fa-solid fa-cart-shopping"></span>
  Checkout
</button>

    </div>
  </div>
</div>
    </header>

    <aside v-if="showProduct" class="sidebar">
  <div class="sidebar-content">
    <h4>Sort by:</h4>
    <div class="sort-container">
      <div v-for="(attribute, index) in ['subject', 'location', 'price', 'availableInventory']" :key="index">
        <input type="checkbox" :id="attribute" :value="attribute" v-model="sortAttributes" class="sort-checkbox">
        <label :for="attribute" class="sort-label">{{ attribute }}</label>
      </div>
    </div>

    <div class="ascending-descending-options">
      <input type="checkbox" id="ascending" v-model="ascendingOrder" class="ascending-checkbox">
      <label for="ascending" class="ascending-label">Ascending</label>

    </div>

  </div>
  <button v-if="showInstallPrompt" @click="promptInstall" class="install-button">Install App</button>
</aside>

    <div v-if="showProduct">
      <LessonComponent :show-product="showProduct" :cart-count="cartCount" :lessons="filteredLessons" :can-add-to-cart="canAddToCart" @add-to-cart="addToCart"/>
    </div>
    <div class="cart" v-else>
      <CheckoutForm :cart-count="cartCount" :order-submitted="orderSubmitted" :save-order="saveOrder" @remove-item-from-cart="removeFromCart" :cart="cart" />
    </div>

    
    
    <!-- Add to Home Screen Button -->
    <button v-if="showInstallPrompt" @click="promptInstall" class="install-button">Install App</button>
    <button class="button cart-toggle-button" @click="toggleCheckout">
  {{ showProduct ? 'View Cart' : 'Back to Lessons' }}
</button>
  </div>
</template>

<script>
import CheckoutForm from './components/CheckoutForm.vue';
import LessonComponent from './components/LessonComponent.vue';

export default {
  name: 'App',
  components: {
    CheckoutForm,
    LessonComponent
  },
  data() {
    return {
      sitename: 'After School Activities',
      products: [],
      cart: [],
      searchQuery: '',
      showProduct: true, // Assuming you want to show products by default
      sortAttributes: [],
      ascendingOrder: false,
      showShoppingCart: false,
      showCheckout: false,
      orderSubmitted: false,
      baseUrl: 'https://cw2-backend-eywd.onrender.com',
      showInstallPrompt: false,
      deferredPrompt: null
    };
  },
  //fetch lessons & pass via prop to LessonComponent
  created() {
    this.fetchLessons();

    window.addEventListener('beforeinstallprompt', (e) => {
      // Prevent the mini-infobar from appearing on mobile
      e.preventDefault();
      // Stash the event so it can be triggered later.
      this.deferredPrompt = e;
      // Update UI to notify the user they can install the PWA
      this.showInstallPrompt = true;
    });
  },
  methods: {
    fetchLessons() {
      fetch(`${this.baseUrl}/lessons`) // Assuming this is the correct endpoint on your server
          .then(response => response.json())
          .then(data => {
            // Handle the response data here (e.g., update Vue.js data)
           this.products = data;
          })
          .catch(error => {
            console.error('Error fetching lessons:', error);
          });
    },
    saveOrder(orderData) {
      fetch(`${this.baseUrl}/new-orders`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(orderData),
      })
          .then(response => response.json())
          .then(data => {
            // Handle the response data here (if needed)
            console.log('Order saved:', data);
            this.orderSubmitted = true;
            setTimeout(function (){
              location.reload()
            },1000)
          })
          .catch(error => {
            console.error('Error saving order:', error);
          });
    },
    addToCart(product) {
      if (this.canAddToCart(product)) {
        const index = this.products.findIndex(p => p.id === product.id);
        if (index !== -1) {
          // Update the original product's available inventory
          this.products[index].availableInventory--;
          // Add a copy of the product to the cart
          this.cart.push({ ...product });
        }
      }
    },
    canAddToCart(product) {
      return product.availableInventory > this.cartCount(product.id) && product.availableInventory > 0;
    },
    cartCount(id) {
      return this.cart.filter(item => item.id === id).length;
    },
    toggleCheckout() {
      this.showProduct = !this.showProduct;
    },
    removeFromCart(productId) {
      const index = this.cart.findIndex(item => item.id === productId);
      if (index !== -1) {
        // Increment the available inventory when removing from the cart
        const productIndex = this.products.findIndex(p => p.id === productId);
        if (productIndex !== -1) {
          this.products[productIndex].availableInventory++;
        }

        //this removes the item from the cart array
        this.cart.splice(index, 1);
      }
    },
    promptInstall() {
      if (this.deferredPrompt) {
        this.deferredPrompt.prompt();
        this.deferredPrompt.userChoice.then((choiceResult) => {
          if (choiceResult.outcome === 'accepted') {
            console.log('User accepted the install prompt');
          } else {
            console.log('User dismissed the install prompt');
          }
          this.deferredPrompt = null;
          this.showInstallPrompt = false;
        });
      }
    }
  },
  computed: {
    filteredLessons() {
      console.log(this.sortAttributes)
      // Check if any sorting attributes are selected
      if (this.sortAttributes.length > 0) {
        // Use a loop to apply sorting for each selected attribute
        this.sortAttributes.forEach(attribute => {
          function compare(a, b) {
            if (a[attribute] > b[attribute]) return 1;
            if (a[attribute] < b[attribute]) return -1;
            return 0;
          }
          // Sort the array based on the selected attribute and order
         this.ascendingOrder ? this.products.sort(compare) : this.products.sort(compare).reverse();
        });
      }
      if (this.searchQuery.length > 1) {
        return this.products.filter(product =>
            (product.title && product.title.toLowerCase().includes(this.searchQuery)) ||
            (product.Location && product.Location.toLowerCase().includes(this.searchQuery))
        );
      }

      return this.products;
    },
  }
};
</script>

<style scoped>
.title-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #333; /* Adjust background color */
  color: #fff; /* Text color */
font-family: 'Times New Roman', Times, serif;
  padding: 10px; /* Adjust padding */
}

.title-bar h1 {
  margin: 0; /* Remove default margin */
}

.search-sort {
  display: flex;
  align-items: center;
}

.search-container {
  margin-right: 10px; /* Adjust margin between search and cart */
}

.sort-checkout-container {
  display: flex;
  align-items: center;
}

.button {
  display: flex;
  align-items: center;
  padding: 10px 15px; /* Adjust padding */
  background-color: #85B96E; /* Button background color */
  color: #fff; /* Button text color */
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.button:hover {
  background-color: #6a9557; /* Adjust hover background color */
}

.fa-solid.fa-cart-shopping {
  margin-left: 5px; /* Adjust icon margin */
}




.sidebar {
  position: fixed; /* Fix sidebar to the viewport */
  top: 70px; /* Adjust based on your header height */
  left: 0; /* Position sidebar on the left */
  width: 170px; /* Adjust sidebar width */
  background-color: #ffffff; /* Background color */
  padding: 20px; /* Padding inside the sidebar */
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1); /* Shadow effect */
  transition: transform 0.3s ease-in-out; /* Smooth transition for sliding effect */
  z-index: 1000; /* Ensure sidebar stays above other content */
  font-family:'Times New Roman', Times, serif;
  color: rgb(0, 0, 0);
}

.sidebar-content {
  margin-bottom: 20px; /* Margin bottom for spacing */
}

.sort-container {
  margin-bottom: 10px; /* Margin bottom for sorting options */
}

.sort-label {
  margin-left: 5px; /* Adjust spacing between checkbox and label */
}

.sort-checkbox {
  margin-right: 5px; /* Adjust spacing between checkboxes */
}

.ascending-descending-options {
  display: flex;
  align-items: center;
}

.ascending-checkbox {
  margin-right: 5px; /* Adjust spacing between checkbox and label */
}

.ascending-label {
  margin-left: 5px; /* Adjust spacing between checkbox and label */
}

/* Hover effect for checkboxes and labels */
input[type="checkbox"]:hover + label {
  color: #007bff; /* Change label text color on hover */
  cursor: pointer; /* Change cursor to pointer on hover */
}

.search-container {
  position: relative; /* Ensure relative positioning for child elements */
  margin-right: 10px; /* Adjust margin as needed */
}

.search-container input[type="text"] {
  width: 300px; /* Adjust width based on your design */
  padding: 10px; /* Add padding for better user interaction */
  font-size: 1rem; /* Adjust font size */
  border: 1px solid #ccc; /* Basic border */
  border-radius: 5px; /* Rounded corners */
  outline: none; /* Remove default input outline */
  transition: border-color 0.3s ease-in-out; /* Smooth transition for border color */
}

.search-container input[type="text"]:focus {
  border-color: #85B96E; /* Change border color on focus */
}

.search-container input[type="text"]::placeholder {
  color: #999; /* Placeholder text color */
}



</style>
