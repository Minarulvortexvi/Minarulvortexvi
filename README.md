// App.jsx - Main application component
import React, { useState, useEffect } from 'react';
import NavBar from './components/NavBar';
import HomePage from './pages/HomePage';
import CartPage from './pages/CartPage';
import { CartProvider } from './context/CartContext';
import './styles/globals.css';

function App() {
  const [currentPage, setCurrentPage] = useState('home');
  
  // Simple routing
  const renderPage = () => {
    switch(currentPage) {
      case 'cart':
        return <CartPage onNavigate={setCurrentPage} />;
      case 'home':
      default:
        return <HomePage onNavigate={setCurrentPage} />;
    }
  };

  return (
    <CartProvider>
      <div className="min-h-screen bg-gray-50 dark:bg-gray-900">
        <NavBar onNavigate={setCurrentPage} />
        <main>
          {renderPage()}
        </main>
        <footer className="bg-white dark:bg-gray-800 border-t py-8 mt-12">
          <div className="container mx-auto px-4">
            <p className="text-center text-gray-500">© 2025 BongoBazaar. All rights reserved.</p>
          </div>
        </footer>
      </div>
    </CartProvider>
  );
}

export default App;

// context/CartContext.jsx - Cart context provider
import React, { createContext, useState, useContext, useEffect } from 'react';
import { products } from '../data/products';

// Create context
export const CartContext = createContext();

// Custom hook for using cart context
export const useCart = () => useContext(CartContext);

// Cart provider component
export const CartProvider = ({ children }) => {
  const [language, setLanguage] = useState('en');
  const [cart, setCart] = useState([]);
  const [wishlist, setWishlist] = useState([]);
  const [quickViewProduct, setQuickViewProduct] = useState(null);
  const [searchQuery, setSearchQuery] = useState('');
  const [searchResults, setSearchResults] = useState([]);
  
  // Load data from localStorage on initial render
  useEffect(() => {
    try {
      const savedCart = localStorage.getItem('bongoBazaarCart');
      const savedWishlist = localStorage.getItem('bongoBazaarWishlist');
      const savedLanguage = localStorage.getItem('bongoBazaarLanguage');
      
      if (savedCart) setCart(JSON.parse(savedCart));
      if (savedWishlist) setWishlist(JSON.parse(savedWishlist));
      if (savedLanguage) setLanguage(savedLanguage);
    } catch (error) {
      console.error('Error loading data from localStorage:', error);
    }
  }, []);
  
  // Save data to localStorage whenever it changes
  useEffect(() => {
    try {
      localStorage.setItem('bongoBazaarCart', JSON.stringify(cart));
      localStorage.setItem('bongoBazaarWishlist', JSON.stringify(wishlist));
      localStorage.setItem('bongoBazaarLanguage', language);
    } catch (error) {
      console.error('Error saving data to localStorage:', error);
    }
  }, [cart, wishlist, language]);
  
  // Search functionality
  const handleSearch = (query) => {
    setSearchQuery(query);
    
    if (!query.trim()) {
      setSearchResults([]);
      return;
    }
    
    // Filter products based on search query
    const filteredProducts = products.filter(product => 
      product.title.en.toLowerCase().includes(query.toLowerCase()) || 
      product.title.bn.includes(query)
    );
    
    setSearchResults(filteredProducts.slice(0, 5)); // Limit to 5 results
  };
  
  // Add to cart
  const addToCart = (product, quantity = 1, variation) => {
    // Default to first variation if none specified
    const selectedVariation = variation || {
      color: product.variations[0].color,
      size: product.variations[0].size
    };
    
    // Check if item with same variation exists
    const existingItemIndex = cart.findIndex(
      item => item.product.id === product.id && 
      JSON.stringify(item.variation) === JSON.stringify(selectedVariation)
    );
    
    if (existingItemIndex >= 0) {
      // Update quantity of existing item
      const updatedCart = [...cart];
      const newQuantity = updatedCart[existingItemIndex].quantity + quantity;
      
      // Check stock
      if (newQuantity > product.stock) {
        alert(language === 'bn' 
          ? 'দুঃখিত, পর্যাপ্ত স্টক নেই!' 
          : 'Sorry, not enough stock available!');
        return;
      }
      
      updatedCart[existingItemIndex].quantity = newQuantity;
      setCart(updatedCart);
    } else {
      // Add as new item
      setCart([...cart, { product, quantity, variation: selectedVariation }]);
    }
  };
  
  // Update cart item quantity
  const updateCartQuantity = (productId, newQuantity, variation) => {
    if (newQuantity < 1) return;
    
    const product = products.find(p => p.id === productId);
    if (newQuantity > product.stock) {
      alert(language === 'bn' ? 'দুঃখিত, পর্যাপ্ত স্টক নেই!' : 'Sorry, not enough stock available!');
      return;
    }
    
    setCart(cart.map(item => 
      (item.product.id === productId && 
       JSON.stringify(item.variation) === JSON.stringify(variation)) 
        ? { ...item, quantity: newQuantity }
        : item
    ));
  };
  
  // Remove from cart
  const removeFromCart = (productId, variation) => {
    setCart(cart.filter(item => 
      !(item.product.id === productId && 
        JSON.stringify(item.variation) === JSON.stringify(variation))
    ));
  };
  
  // Clear cart
  const clearCart = () => {
    setCart([]);
  };
  
  // Toggle wishlist
  const toggleWishlist = (product) => {
    if (wishlist.some(item => item.id === product.id)) {
      setWishlist(wishlist.filter(item => item.id !== product.id));
    } else {
      setWishlist([...wishlist, product]);
    }
  };
  
  // Calculate cart totals
  const getCartTotals = () => {
    const itemCount = cart.reduce((total, item) => total + item.quantity, 0);
    
    const subtotal = cart.reduce((total, item) => {
      const price = parseFloat(item.product.price.en.replace(/[^0-9.]/g, ''));
      return total + (price * item.quantity);
    }, 0);
    
    return { itemCount, subtotal };
  };
  
  // Format currency based on language
  const formatCurrency = (amount) => {
    return language === 'bn' 
      ? `৳${amount.toFixed(2).replace('.', ',')}`
      : `$${amount.toFixed(2)}`;
  };
  
  return (
    <CartContext.Provider value={{
      language,
      setLanguage,
      cart,
      wishlist,
      quickViewProduct,
      setQuickViewProduct,
      searchQuery,
      searchResults,
      handleSearch,
      addToCart,
      updateCartQuantity,
      removeFromCart,
      clearCart,
      toggleWishlist,
      getCartTotals,
      formatCurrency
    }}>
      {children}
    </CartContext.Provider>
  );
};

// components/NavBar.jsx - Navigation component
import React, { useState } from 'react';
import { Search, Heart, ShoppingCart, Menu, X } from 'lucide-react';
import { 
  Input, 
  Button, 
  ScrollArea, 
  Sheet,
  SheetContent,
  SheetTrigger,
  Badge,
  Drawer,
  DrawerContent,
  DrawerTrigger
} from '@/components/ui';
import { useCart } from '../context/CartContext';

// Navigation links data
const navLinks = [
  { key: 'home', text: { en: 'Home', bn: 'হোম' } },
  { key: 'products', text: { en: 'Products', bn: 'পণ্যসমূহ' } },
  { key: 'categories', text: { en: 'Categories', bn: 'বিভাগসমূহ' } },
  { key: 'deals', text: { en: 'Deals', bn: 'অফার' } },
  { key: 'about', text: { en: 'About', bn: 'আমাদের সম্পর্কে' } },
  { key: 'contact', text: { en: 'Contact', bn: 'যোগাযোগ' } }
];

const NavBar = ({ onNavigate }) => {
  const {
    language,
    setLanguage,
    cart,
    wishlist,
    searchQuery,
    searchResults,
    handleSearch,
    setQuickViewProduct,
    addToCart,
    removeFromCart,
    toggleWishlist,
    getCartTotals,
    formatCurrency
  } = useCart();
  
  const [mobileMenuOpen, setMobileMenuOpen] = useState(false);
  const { itemCount, subtotal } = getCartTotals();

  return (
    <header className="sticky top-0 z-50 w-full bg-white shadow-md dark:bg-gray-800">
      <div className="container mx-auto px-4">
        <div className="flex h-16 items-center justify-between">
          <div className="flex items-center">
            <a 
              href="#"
              onClick={(e) => {
                e.preventDefault();
                onNavigate('home');
              }} 
              className="text-2xl font-bold tracking-tight"
            >
              BongoBazaar
            </a>
            
            {/* Desktop Navigation */}
            <nav className="ml-8 hidden md:flex space-x-6">
              {navLinks.map((link) => (
                <a 
                  key={link.key} 
                  href="#"
                  onClick={(e) => {
                    e.preventDefault();
                    onNavigate(link.key);
                  }}
                  className="text-sm font-medium transition-colors hover:text-blue-600 dark:hover:text-blue-400"
                >
                  {link.text[language]}
                </a>
              ))}
            </nav>
          </div>
          
          <div className="flex items-center space-x-4">
            {/* Search */}
            <div className="relative hidden md:flex w-64">
              <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-gray-500 dark:text-gray-400" />
              <Input
                type="search"
                placeholder={language === "bn" ? "খুঁজুন..." : "Search..."}
                className="pl-8"
                value={searchQuery}
                onChange={(e) => handleSearch(e.target.value)}
              />
              
              {/* Search results dropdown */}
              {searchResults.length > 0 && searchQuery && (
                <div className="absolute top-full left-0 right-0 mt-1 bg-white dark:bg-gray-800 shadow-lg rounded-md overflow-hidden z-50">
                  <div className="p-2">
                    <h3 className="text-sm font-semibold mb-2">
                      {language === "bn" ? "সার্চ রেজাল্ট" : "Search Results"}
                    </h3>
                    <ScrollArea className="h-64">
                      {searchResults.map(product => (
                        <div 
                          key={product.id}
                          className="flex items-center gap-3 p-2 hover:bg-gray-100 dark:hover:bg-gray-700 rounded cursor-pointer"
                          onClick={() => {
                            setQuickViewProduct(product);
                            handleSearch("");
                          }}
                        >
                          <img 
                            src={product.image} 
                            alt={product.title[language]} 
                            className="w-12 h-12 object-cover rounded" 
                          />
                          <div>
                            <div className="font-medium">{product.title[language]}</div>
                            <div className="text-sm text-blue-600 dark:text-blue-400">{product.price[language]}</div>
                          </div>
                        </div>
                      ))}
                    </ScrollArea>
                  </div>
                </div>
              )}
            </div>
            
            {/* Language switcher */}
            <div className="flex items-center space-x-1">
              <Button 
                variant={language === "bn" ? "default" : "outline"} 
                size="sm" 
                onClick={() => setLanguage("bn")}
                className="h-8 rounded-full px-3"
              >
                বাংলা
              </Button>
              <Button 
                variant={language === "en" ? "default" : "outline"} 
                size="sm" 
                onClick={() => setLanguage("en")}
                className="h-8 rounded-full px-3"
              >
                ENG
              </Button>
            </div>
            
            {/* Wishlist */}
            <Drawer>
              <DrawerTrigger asChild>
                <Button variant="ghost" size="icon" className="relative">
                  <Heart className="h-5 w-5" />
                  {wishlist.length > 0 && (
                    <Badge className="absolute -top-1 -right-1 h-5 w-5 p-0 flex items-center justify-center">
                      {wishlist.length}
                    </Badge>
                  )}
                </Button>
              </DrawerTrigger>
              <DrawerContent>
                <div className="p-4">
                  <h2 className="text-lg font-semibold mb-4">
                    {language === "bn" ? "পছন্দের তালিকা" : "Wishlist"}
                  </h2>
                  
                  {wishlist.length === 0 ? (
                    <div className="text-center py-8">
                      <Heart className="h-12 w-12 mx-auto text-gray-300 dark:text-gray-600" />
                      <p className="mt-2 text-gray-500">
                        {language === "bn" ? "আপনার পছন্দের তালিকা খালি" : "Your wishlist is empty"}
                      </p>
                    </div>
                  ) : (
                    <ScrollArea className="h-96">
                      {wishlist.map(item => (
                        <div key={item.id} className="flex items-center gap-3 p-3 border-b">
                          <img 
                            src={item.image} 
                            alt={item.title[language]} 
                            className="w-16 h-16 object-cover rounded" 
                          />
                          <div className="flex-1">
                            <h3 className="font-medium">{item.title[language]}</h3>
                            <p className="text-blue-600 dark:text-blue-400">{item.price[language]}</p>
                          </div>
                          <div className="flex gap-2">
                            <Button 
                              variant="outline" 
                              size="sm"
                              onClick={() => addToCart(item)}
                            >
                              <ShoppingCart className="h-4 w-4 mr-1" />
                              {language === "bn" ? "কার্টে যোগ করুন" : "Add to Cart"}
                            </Button>
                            <Button 
                              variant="ghost" 
                              size="sm"
                              onClick={() => toggleWishlist(item)}
                            >
                              <X className="h-4 w-4" />
                            </Button>
                          </div>
                        </div>
                      ))}
                    </ScrollArea>
                  )}
                </div>
              </DrawerContent>
            </Drawer>
            
            {/* Shopping Cart */}
            <Drawer>
              <DrawerTrigger asChild>
                <Button 
                  variant="ghost" 
                  size="icon" 
                  className="relative"
                  onClick={() => {
                    // When you want to navigate to cart page instead of showing drawer
                    // onNavigate('cart'); 
                  }}
                >
                  <ShoppingCart className="h-5 w-5" />
                  {itemCount > 0 && (
                    <Badge className="absolute -top-1 -right-1 h-5 w-5 p-0 flex items-center justify-center">
                      {itemCount}
                    </Badge>
                  )}
                </Button>
              </DrawerTrigger>
              <DrawerContent>
                <div className="p-4">
                  <h2 className="text-lg font-semibold mb-4">
                    {language === "bn" ? "শপিং কার্ট" : "Shopping Cart"}
                  </h2>
                  
                  {cart.length === 0 ? (
                    <div className="text-center py-8">
                      <ShoppingCart className="h-12 w-12 mx-auto text-gray-300 dark:text-gray-600" />
                      <p className="mt-2 text-gray-500">
                        {language === "bn" ? "আপনার কার্ট খালি" : "Your cart is empty"}
                      </p>
                    </div>
                  ) : (
                    <>
                      <ScrollArea className="h-72">
                        {cart.map(item => (
                          <div key={`${item.product.id}-${JSON.stringify(item.variation)}`} className="flex items-center gap-3 p-3 border-b">
                            <img 
                              src={item.product.image} 
                              alt={item.product.title[language]} 
                              className="w-16 h-16 object-cover rounded" 
                            />
                            <div className="flex-1">
                              <h3 className="font-medium">{item.product.title[language]}</h3>
                              <div className="text-sm text-gray-500 mt-1">
                                {item.variation.color && (
                                  <span>
                                    {language === 'bn' ? 'রঙ' : 'Color'}: {item.variation.color}
                                  </span>
                                )}
                                {item.variation.size && (
                                  <span className="ml-3">
                                    {language === 'bn' ? 'সাইজ' : 'Size'}: {item.variation.size}
                                  </span>
                                )}
                              </div>
                            </div>
                            <div>
                              <p className="text-blue-600 dark:text-blue-400 font-medium">
                                {formatCurrency(parseFloat(item.product.price.en.replace(/[^0-9.]/g, '')) * item.quantity)}
                              </p>
                              <Button 
                                variant="ghost" 
                                size="sm" 
                                className="p-0 h-6 mt-1"
                                onClick={() => removeFromCart(item.product.id, item.variation)}
                              >
                                <X className="h-3 w-3 mr-1" />
                                {language === "bn" ? "মুছুন" : "Remove"}
                              </Button>
                            </div>
                          </div>
                        ))}
                      </ScrollArea>
                      
                      <div className="mt-4 border-t pt-4">
                        <div className="flex justify-between mb-4">
                          <span className="font-semibold">{language === "bn" ? "মোট মূল্য" : "Total Price"}</span>
                          <span className="font-semibold text-blue-600 dark:text-blue-400">
                            {formatCurrency(subtotal)}
                          </span>
                        </div>
                        <div className="flex space-x-2">
                          <Button 
                            className="flex-1"
                            onClick={() => {
                              onNavigate('cart');
                            }}
                          >
                            {language === "bn" ? "কার্ট দেখুন" : "View Cart"}
                          </Button>
                          <Button className="flex-1">
                            {language === "bn" ? "চেকআউট" : "Checkout"}
                          </Button>
                        </div>
                      </div>
                    </>
                  )}
                </div>
              </DrawerContent>
            </Drawer>
            
            {/* Mobile menu button */}
            <Sheet open={mobileMenuOpen} onOpenChange={setMobileMenuOpen}>
              <SheetTrigger asChild>
                <Button variant="ghost" size="icon" className="md:hidden">
                  <Menu className="h-5 w-5" />
                </Button>
              </SheetTrigger>
              <SheetContent side="left">
                <div className="py-4">
                  <a 
                    href="#"
                    onClick={(e) => {
                      e.preventDefault();
                      onNavigate('home');
                      setMobileMenuOpen(false);
                    }}
                    className="text-xl font-bold tracking-tight block mb-6"
                  >
                    BongoBazaar
                  </a>
                  
                  <nav className="flex flex-col space-y-4">
                    {navLinks.map((link) => (
                      <a 
                        key={link.key} 
                        href="#"
                        onClick={(e) => {
                          e.preventDefault();
                          onNavigate(link.key);
                          setMobileMenuOpen(false);
                        }}
                        className="text-sm font-medium transition-colors hover:text-blue-600 dark:hover:text-blue-400"
                      >
                        {link.text[language]}
                      </a>
                    ))}
                  </nav>
                  
                  {/* Mobile search */}
                  <div className="relative mt-6">
                    <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-gray-500 dark:text-gray-400" />
                    <Input
                      type="search"
                      placeholder={language === "bn" ? "খুঁজুন..." : "Search..."}
                      className="pl-8"
                      value={searchQuery}
                      onChange={(e) => handleSearch(e.target.value)}
                    />
                  </div>
                </div>
              </SheetContent>
            </Sheet>
          </div>
        </div>
      </div>
    </header>
  );
};

export default NavBar;

// pages/CartPage.jsx - Cart page component
import React, { useState } from 'react';
import { ArrowLeft, Trash, Plus, Minus, RefreshCw, Truck, Shield, CreditCard } from 'lucide-react';
import { 
  Button, 
  Input, 
  Select, 
  SelectContent, 
  SelectItem, 
  SelectTrigger, 
  SelectValue,
  Card,
  CardContent,
  Separator,
  Alert,
  AlertDescription,
  AlertTitle
} from '@/components/ui';
import { useCart } from '../context/CartContext';

const CartPage = ({ onNavigate }) => {
  const {
    language,
    cart,
    updateCartQuantity,
    removeFromCart,
    clearCart,
    formatCurrency,
    getCartTotals,
    addToCart
  } = useCart();
  
  const [promoCode, setPromoCode] = useState('');
  const [promoApplied, setPromoApplied] = useState(false);
  const [discount, setDiscount] = useState(0);
  const [shippingMethod, setShippingMethod] = useState('standard');
  const [shippingCost, setShippingCost] = useState(5.99);
  
  const { subtotal } = getCartTotals();

  // Apply promo code
  const applyPromoCode = () => {
    // Simple promo code logic
    if (promoCode.toUpperCase() === 'BONGO20') {
      setDiscount(subtotal * 0.2); // 20% discount
      setPromoApplied(true);
    } else if (promoCode.toUpperCase() === 'FREESHIP') {
      setShippingCost(0);
      setPromoApplied(true);
    } else {
      alert(language === 'bn' ? 'অবৈধ প্রোমো কোড!' : 'Invalid promo code!');
      setPromoApplied(false);
      setDiscount(0);
      // Reset shipping cost
      if (shippingMethod === 'standard') setShippingCost(5.99);
      if (shippingMethod === 'express') setShippingCost(12.99);
    }
  };

  // Handle shipping method change
  const handleShippingChange = (value) => {
    setShippingMethod(value);
    if (value === 'standard') setShippingCost(5.99);
    if (value === 'express') setShippingCost(12.99);
    if (value === 'pickup') setShippingCost(0);
  };

  // Calculate total
  const calculateTotal = () => {
    return (subtotal - discount) + shippingCost;
  };

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="flex items-center mb-6">
        <Button 
          variant="ghost"
          className="inline-flex items-center text-blue-600 hover:text-blue-800"
          onClick={() => onNavigate('home')}
        >
          <ArrowLeft className="h-4 w-4 mr-1" />
          {language === 'bn' ? 'শপিং চালিয়ে যান' : 'Continue Shopping'}
        </Button>
        <h1 className="text-2xl font-bold ml-4">
          {language === 'bn' ? 'শপিং কার্ট' : 'Shopping Cart'}
        </h1>
      </div>

      {cart.length === 0 ? (
        <div className="text-center py-12">
          <div className="bg-gray-100 rounded-full h-24 w-24 flex items-center justify-center mx-auto mb-4">
            <RefreshCw className="h-10 w-10 text-gray-400" />
          </div>
          <h2 className="text-xl font-semibold mb-2">
            {language === 'bn' ? 'আপনার কার্ট খালি আছে' : 'Your cart is empty'}
          </h2>
          <p className="text-gray-500 mb-6">
            {language === 'bn' 
              ? 'শপিং শুরু করতে আমাদের দোকানে যান' 
              : 'Visit our shop to start adding items to your cart'}
          </p>
          <Button 
            size="lg" 
            className="px-8"
            onClick={() => onNavigate('home')}
          >
            {language === 'bn' ? 'দোকানে যান' : 'Go to Shop'}
          </Button>
        </div>
      ) : (
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          {/* Cart Items */}
          <div className="lg:col-span-2">
            <div className="bg-white rounded-lg shadow-sm border p-6">
              <div className="flex justify-between mb-4">
                <h2 className="text-lg font-semibold">
                  {language === 'bn' ? 'কার্ট আইটেম' : 'Cart Items'} ({cart.reduce((total, item) => total + item.quantity, 0)})
                </h2>
                <Button 
                  variant="ghost" 
                  size="sm" 
                  className="text-red-500 hover:text-red-700" 
                  onClick={clearCart}
                >
                  <Trash className="h-4 w-4 mr-1" />
                  {language === 'bn' ? 'সব মুছুন' : 'Clear All'}
                </Button>
              </div>

              {cart.map((item, index) => (
                <div key={`${item.product.id}-${JSON.stringify(item.variation)}`} className="py-4">
                  {index > 0 && <Separator className="mb-4" />}
                  <div className="flex">
                    <div className="flex-shrink-0 w-24 h-24 bg-gray-100 rounded overflow-hidden">
                      <img 
                        src={item.product.image} 
                        alt={item.product.title[language]} 
                        className="w-full h-full object-cover"
                      />
                    </div>
                    <div className="flex-1 ml-4">
                      <div className="flex justify-between">
                        <div>
                          <h3 className="font-medium">{item.product.title[language]}</h3>
                          <div className="text-sm text-gray-500 mt-1">
                            {item.variation.color && (
                              <span>
                                {language === 'bn' ? 'রঙ' : 'Color'}: {item.variation.color}
                              </span>
                            )}
                            {item.variation.size && (
                              <span className="ml-3">
                                {language === 'bn' ? 'সাইজ' : 'Size'}: {item.variation.size}
                              </span>
                            )}
                          </div>
                          <div className="text-blue-600 font-medium mt-2">
                            {formatCurrency(parseFloat(item.product.price.en.replace(/[^0-9.]/g, '')) * item.quantity)}
                          </div>
                        </div>
                        <div>
                          <Button 
                            variant="ghost" 
                            size="sm" 
                            className="text-red-500 hover:text-red-700 p-0 h-6"
                            onClick={() => removeFromCart(item.product.id, item.variation)}
                          >
                            <Trash className="h-4 w-4" />
                          </Button>
                        </div>
                      </div>
                      <div className="flex items-center mt-3">
                        <div className="flex items-center border rounded">
                          <Button
