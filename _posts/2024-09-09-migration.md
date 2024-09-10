---
layout: post
title: First migration file and database
author: Hamza Gedikkaya
categories: E-Commerce
excerpt_image: /assets/images/post_imgs/firstMigration.gif
banner:
  image: /assets/images/post_imgs/firstMigrationBanner.jpg
  height: "50vh"
tags: Ruby Rails E-Commerce
top: 2
---

## E-commerce Application Database Migration

This migration defines the structure of a database for a comprehensive e-commerce platform. The platform allows users to browse products, manage their favorites, add products to a shopping cart, make orders, follow stores, and leave reviews for both products and stores. Additionally, store owners can manage their products and receive inquiries from customers. The migration also supports a discount coupon system, enabling stores to offer promotions on specific categories, products, or stores.

## Database Tables Overview

This migration includes the following tables:

- **Users**: Stores user details including name, email, phone number, gender, and date of birth. Passwords are securely stored using a `password_digest`.
  
- **Cities & Districts**: Represents geographical locations, allowing users to select their city and district for delivery addresses.

- **Addresses**: Users can save multiple addresses with attributes like address type, postal code, and default status. These addresses are linked to the user, city, and district.

- **Categories & Subcategories**: Organizes products into hierarchical categories and subcategories, making it easier for users to browse through the product catalog.

- **Brands**: Represents the various brands available in the platform, linked to products.

- **Products**: Contains information about each product, including its subcategory, brand, name, and description.

- **Attributes**: Allows products to have customizable attributes like color, size, or material, stored as JSONB for flexibility.

- **Stores**: Stores represent sellers in the system, each with contact information and location details.

- **Store Products**: Defines the products available in each store, including stock levels and pricing.

- **Last Viewed**: Tracks the last products viewed by users for better personalization and user experience.

- **Favorites**: Allows users to save their favorite products for quick access.

- **Shopping Cart**: Stores the products that users add to their shopping cart for eventual checkout.

- **Orders**: Stores order details for users, including the total amount, status, and timestamps for shipping and completion.

- **Store Follows**: Allows users to follow their favorite stores and stay updated on new products or promotions.

- **Order Items**: Contains details about the products within an order, including the quantity and price of each item.

- **Product Inquiries**: Enables users to ask questions about products directly to stores and receive responses.

- **Product & Store Reviews**: Users can leave ratings and reviews for both products and stores, providing feedback for future customers.

- **Discount Coupons**: Allows stores to offer promotions with discount codes. These codes can apply to specific categories, stores, or products, and have configurable start and end dates.

## Migration Code

```ruby
class FirstMigration < ActiveRecord::Migration[7.2]
  def change
    create_table :users do |t|
      t.string :name_surname
      t.string :email, null: false
      t.string :gsm
      t.string :password_digest
      t.integer :gender
      t.date :date_of_birth
      t.timestamps
    end

    create_table :cities do |t|
      t.string :name
      t.timestamps
    end

    create_table :districts do |t|
      t.string :name
      t.references :city, null: false, foreign_key: true
      t.timestamps
    end

    create_table :addresses do |t|
      t.references :user, null: false, foreign_key: true
      t.string :line_1
      t.string :line_2
      t.string :postal_code
      t.string :phone_number
      t.integer :address_type
      t.boolean :is_default, default: false
      t.references :city, null: false, foreign_key: true
      t.references :district, null: false, foreign_key: true
      t.timestamps
    end

    create_table :categories do |t|
      t.string :name
      t.timestamps
    end

    create_table :subcategories do |t|
      t.references :category, null: false, foreign_key: true
      t.string :name
      t.timestamps
    end

    create_table :brands do |t|
      t.string :name
      t.timestamps
    end

    create_table :products do |t|
      t.references :subcategory, null: false, foreign_key: true
      t.references :brand, null: false, foreign_key: true
      t.string :name
      t.text :description
      t.timestamps
    end

    create_table :attributes do |t|
      t.string :name
      t.jsonb :values
      t.timestamps
    end

    create_table :product_attributes do |t|
      t.references :product, null: false, foreign_key: true
      t.references :attribute, null: false, foreign_key: true
      t.timestamps
    end

    create_table :stores do |t|
      t.string :name
      t.text :description
      t.string :contact_mail
      t.string :contact_phone
      t.string :location
      t.timestamps
    end

    create_table :store_products do |t|
      t.references :store, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :stock, default: 0
      t.decimal :price, precision: 10, scale: 2
      t.timestamps
    end

    create_table :last_viewed do |t|
      t.references :user, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.timestamps
    end

    create_table :favorites do |t|
      t.references :user, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.timestamps
    end

    create_table :shopping_cart do |t|
      t.references :user, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.timestamps
    end

    create_table :orders do |t|
      t.references :user, null: false, foreign_key: true
      t.decimal :total_amount, precision: 10, scale: 2
      t.string :status
      t.datetime :shipped_at
      t.datetime :completed_at
      t.timestamps
    end

    create_table :store_follows do |t|
      t.references :user, null: false, foreign_key: true
      t.references :store, null: false, foreign_key: true
      t.timestamps
    end

    create_table :order_items do |t|
      t.references :order, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :quantity
      t.decimal :price, precision: 10, scale: 2
      t.timestamps
    end

    create_table :product_inquiries do |t|
      t.references :user, null: false, foreign_key: true
      t.references :store, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.text :question
      t.text :response
      t.boolean :is_responded, default: false
      t.timestamps
    end

    create_table :product_reviews do |t|
      t.references :user, null: false, foreign_key: true
      t.references :product, null: false, foreign_key: true
      t.integer :rating
      t.text :review
      t.timestamps
    end

    create_table :store_reviews do |t|
      t.references :user, null: false, foreign_key: true
      t.references :store, null: false, foreign_key: true
      t.integer :rating
      t.text :review
      t.timestamps
    end

    create_table :discount_coupons do |t|
      t.string :code, null: false
      t.text :description
      t.decimal :discount_amount, precision: 10, scale: 2
      t.decimal :discount_percentage, precision: 5, scale: 2
      t.datetime :start_date, null: false
      t.datetime :end_date, null: false
      t.jsonb :applicable_categories, default: []
      t.jsonb :applicable_stores, default: []
      t.jsonb :applicable_products, default: []
      t.timestamps
    end
  end
end
```


## UML Diagram

Additionally, you can review the visual representation of the UML diagram for this migration below.

![uml_diagram](/assets/images/post_imgs/firstMigrationUML.png)
