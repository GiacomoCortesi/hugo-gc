---
title: "Build an e-commerce website with Go and React"
date: "2024-02-19T17:15:00+01:00"
slug: build-e-commerce-website-with-go-and-react
category: blog 
summary:
description: 
cover:
  image:
  alt:
  caption: 
  relative: true
showtoc: true
draft: false
---

# Overview

This article provides an in-depth overview about designing and implementing an e-commerce
website application with a golang backend and a react frontend.

Check [here](https://github.com/GiacomoCortesi/ccac) the full code.

View the resulting website at [couscousacolazione.com](https://couscousacolazione.com/)

With my band, CousCous a colazione we made the merchandise, so I took the chance
to build up a website with e-commerce functionality.

The backend is implemented in go, leveraging the gin framework and the gorm library
for database access.

The chosen database is mongoDB, a non-relational database that handles data as BSON
documents, a binary JSON format representation.

Moving to a different storage backend is a matter of creating the proper database
go repo package and inject it into the services.

The frontend uses react, having worked both with react and angular in the past I
tend to like more the former as it is just a javascript library and not a complete
framework. I like to have the flexibility to make my own choices and add things only
when they are needed.

NGINX is therefore used as a reverse proxy to serve both the website and expose the 
REST API.

The frontend dockers include the generation/renewal of let's encrypt certificates 
in order to serve the website safely over https, completely for free.

# Data model and API definition
Having no experience about e-commerce websites development I just started looking
around some popular e-commerce websites to grab some insights.

I decided to keep things simple and reduce the user interactions to the following:
- the user must be able to navigate and view all the products
- the user must be able to add a product to the cart
- the user must be able to remove a product from the cart
- the user must be able to view its own cart
- the user must be able to place an order of the current cart

I didn't want to go through the full user account registration and
management, to keep things simple and have a smoother buying experience. Though,
we need to keep track of the user somehow as we must handle the user cart.
Since we are using gin, we have a nice 
[sessions middleware package](https://github.com/gin-contrib/sessions) 
at our disposal.

On one hand I wanted to keep things simple, on the other hand I wanted something
production-ready that could be easily expanded to introduce additional features.
So I modeled the product item as an object identified by its Stock Keeping Unit
(SKU), each product object in turns may contain a non-empty array of product
variations. A product variation is a product object by itself and has its own SKU.
For instance, for a t-shirt product, the variations will include the different sizes,
colors, etc.

I wanted to take care of the warehouse management in order to be able to add some admin portal
functionality in the future, in which the e-commerce administrator can login and
manage the products warehouse adding/removing/modifying products, and tracking orders.

The database is made of just three collections (i.e. a series of JSON documents):
 - order: models the user order for making purchases
 - cart: models the user cart
 - product: models the e-commerce products

The product struct:
```
type Product struct {
  ID   ID     `yaml:"id"  json:"id"  bson:"_id,omitempty"`
  Type string `yaml:"type"  json:"type"  bson:"type"`

  StockKeepingUnit `yaml:",inline"  json:",inline"  bson:"inline"`
  Variations       []StockKeepingUnit `yaml:"variations"  json:"variations"  bson:"variations,omitempty"`
}

type StockKeepingUnit struct {
    Sku         string   `yaml:"sku"  json:"sku"  bson:"sku,omitempty"`
    Quantity    int      `yaml:"quantity"  json:"quantity"  bson:"quantity,omitempty"`
    Options     Options  `yaml:"options"  json:"options"  bson:"options,omitempty"`
    Price       Price    `yaml:"price"  json:"price"  bson:"price"`
    Rating      float32  `yaml:"rating"  json:"rating"  bson:"rating,omitempty"`
    Categories  []string `yaml:"categories"  json:"categories"  bson:"categories,omitempty"`
    Images      []string `yaml:"images"  json:"images"  bson:"images,omitempty"`
    Title       string   `yaml:"title"  json:"title"  bson:"title,omitempty"`
    Description string   `yaml:"description"  json:"description"  bson:"description,omitempty"`
}

type Options struct {
    Color           string   `yaml:"color"  json:"color"  bson:"color,omitempty"`
    AvailableColors []string `yaml:"available_colors"  json:"available_colors"  bson:"available_colors,omitempty"`
    Size            string   `yaml:"size"  json:"size"  bson:"size,omitempty"`
    AvailableSizes  []string `yaml:"available_sizes"  json:"available_sizes"  bson:"available_sizes,omitempty"`
}

type Price struct {
    Value    decimal.Decimal `yaml:"value"  json:"value"  bson:"value,omitempty"`
    Currency string          `yaml:"currency"  json:"currency"  bson:"currency,omitempty"`
}
```

The cart struct
```
type Cart struct {
	ID              ID              `json:"id" bson:"_id,omitempty"`
	Items           []CartItem      `json:"items" bson:"items"`
	Token           string          `json:"token" bson:"token"`
	LastModified    time.Time       `json:"-" bson:"last_modified"`
	Total           Price           `json:"total" bson:"total"`
	ShippingOptions ShippingOptions `json:"shipping_options"`
}
type CartItem struct {
	SKU       string `json:"sku" bson:"sku"`
	Quantity  int    `json:"quantity" bson:"quantity"`
	ProductID ID     `json:"product_id" bson:"product_id"`
	Total     Price  `json:"total" bson:"total"`
}
```

The order struct:
```
type Order struct {
	ID          ID          `json:"id" bson:"_id,omitempty"`
	Cart        Cart        `json:"cart"`
	Status      OrderStatus `json:"status"`
	Date        time.Time   `json:"date"`
	LastUpdated time.Time   `json:"last_updated"`
	Completed   bool        `json:"completed"`
	Shipping    Shipping    `json:"shipping"`
	Total       Price       `json:"total"`

	// paypal specific parameters
	PaypalOrderID string `json:"paypal_order_id"`
}

type Shipping struct {
	Method      string `json:"method"`
	Cost        Price  `json:"cost"`
	Title       string `json:"title"`
	Detail      string `json:"detail,omitempty"`
	WorkingDays string `json:"working_days,omitempty"`
	Location    string `json:"location,omitempty"`
}
```

# Handle monetary data
Another thing to take into account when coding an e-commerce website is how to
handle the computations over monetary data.

As you know, float and double can't really be used as the vast majority of
money-like numbers don't have an exact representation as an integer times a power
of 2.

Fortunately, mongodb comes in help with its built in decimal128 data type that
is designed specifically for handling monetary data.
See more in this [mongo db article about modeling monetary data](mongodb.com/docs/manual/tutorial/model-monetary-data/)
