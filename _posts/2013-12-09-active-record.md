---
layout: post
title: Active Record 随笔
categories: Rails 原创
tags: rails active_record
---

## 1 什么是Active Record  

________________________

Active Record是orm框架，在rails框架中处于m（model）的位置。它主要有以下几个功能：  
>1. 表达模型和模型对应的数据  
>2. 表达模型之间的关系
>3. 表达模型层级之间的关系
>4. 可以在数据持久化之前进行数据验证
>5. 按照面向对象的方式进行数据库操作

和rails的经典思想“约定大于配置”一样，Active Record也是采用的这样的实现方式。这样的设计可以降低开发的成本，让开发成员更多的专注在业务逻辑的处理上。

<!--more--> 

## 2 Active Record migration datatypes:
  
* binary  
* boolean  
* date  
* datetime  
* decimal  
* float  
* integer  
* primary_key  
* references  
* string  
* text  
* time    
* timestamp  

*如果使用的是PostgreSQL，还可以使用下面的datatypes*  

* hstore
* array
* cidr_address
* ip_address
* mac_address  

## 3 Specify Active Record Migration Engine

```ruby
  create_table :products, options: "ENGINE=BLACKHOLE" do |t|
     t.string :name, null: false
  end
```
注意： options: "ENGINE=BLACKHOLE"可以指定表类型，例如：mysql默认的数据类型是InnoDB，可以将它指定为MYISAM。

## 4 创建连接表
Migration方法“create_join_table”创建一个HABTM连接表。

```ruby
  create_table :products, options: "ENGINE=BLACKHOLE" do |t|
     t.string :name, null: false
     create_join_table :products, :categories
  end
```
按以上代码migration之后将产生表名为products_categories的表，并有product_id,category_id两个属性。
可以通过：

>create_join_table :products, :categories, table_name: :categorization  
自定义表名。  

>create_join_table :products, :categories, column_options: {null: true}  
默认产生的字段可以为空。

## 5 创建多对多自连接（self join）

```ruby
    has_and_belongs_to_many :followers, 
                  class_name: "User", 
                  join_table: :followers_followings,
                  foreign_key: :following_id,
                  association_foreign_key: :follower_id 
    has_and_belongs_to_many :followings, 
              class_name: "User", 
              join_table: :followers_followings,
              foreign_key: :follower_id,  
              association_foreign_key: :following_id
``` 
注：这种方式可以不用创建模型，如：followers_followings

```ruby
       has_many :from_followers, foreign_key: :follower_id, class_name: "FollowersFollowings"
       has_many :followers, through: :froms_followers 
       has_many :to_followings, foreign_key: :following_id, class_name: "FollowersFollowings"
       has_many :followings, through: :to_followings
```
注：这种方式可以需要创建模型，如：FollowersFollowings


## 6 ActiveRecord Migration Terms

### **1.Polymorphic (多态)**
________________________
  
 ```ruby
    class Picture < ActiveRecord::Base
      belongs_to :imageable, polymorphic: true
    end
     
    class Employee < ActiveRecord::Base
      has_many :pictures, as: :imageable
    end
     
    class Product < ActiveRecord::Base
      has_many :pictures, as: :imageable
    end
 ```  
________________________
  

### **2.has_many :through**  
________________________
  
经常被用作建立多对多的关系。

```ruby  
    class Physician < ActiveRecord::Base
      has_many :appointments
      has_many :patients, through: :appointments
    end
     
    class Appointment < ActiveRecord::Base
      belongs_to :physician
      belongs_to :patient
    end
     
    class Patient < ActiveRecord::Base
      has_many :appointments
      has_many :physicians, through: :appointments
    end
```
________________________

### **3.delegate(委托)**  
________________________
  
```ruby
  belongs_to :exchange  
  delegate :name, :begin_at, :end_at, :pic_url, :type, :pic, :description, :cost, :price,  to: :exchange
```
________________________

### **4.inverse_of**
________________________

主要有三个作用：  

* 在验证的时候用到

```ruby
    class LineItem < ActiveRecord::Base
      belongs_to :order
      validates :order, presence: true
    end
    class Order < ActiveRecord::Base
      has_many :line_items, inverse_of: :order
    end
```

* 确保一对多关系的同步(在内存中只存在一个对象)

```ruby
    class Customer < ActiveRecord::Base
      has_many :orders
    end
     
    class Order < ActiveRecord::Base
      belongs_to :customer
    end
    
    c = Customer.first   
    o = c.orders.first  
    c.first_name == o.customer.first_name # => true  
    c.first_name = 'Manny'  
    c.first_name == o.customer.first_name # => true  
    
    
    
    class Customer < ActiveRecord::Base
      has_many :orders, inverse_of: :customer
    end
     
    class Order < ActiveRecord::Base
      belongs_to :customer, inverse_of: :orders
    end
    
    c = Customer.first
    o = c.orders.first
    c.first_name == o.customer.first_name # => true
    c.first_name = 'Manny'
    c.first_name == o.customer.first_name # => true
```

* 确保counter_cache: true有用

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, dependent: :destroy,
        counter_cache: true
    end
    class customer < ActiveRecord::Base
      has_many :Orders, inverse_of: :customer
    end
```
________________________

### **5.autosave**
________________________

```ruby
    class Post
      has_one :author, autosave: true
    end

    post = Post.find(1)
    post.title       # => "The current global position of migrating ducks"
    post.author.name # => "alloy"
    
    post.title = "On the migration of ducks"
    post.author.name = "Eloy Duran"
    
    post.save
    post.reload
    post.title       # => "On the migration of ducks"
    post.author.name # => "Eloy Duran"
```
________________________

### **6.class_name**
________________________

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, class_name: "Patron"
    end
```
________________________

### **7.counter_cache**
________________________

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, dependent: :destroy,
        counter_cache: true
    end
```
注：会在对应的customer表中自动的产生orders_count字段,并在save和destroy是自动的改变orders_count的值
________________________

### **8.dependent**
________________________

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, dependent: :destroy,
        counter_cache: true
    end
```
________________________

### **9.foreign_key**
________________________

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, class_name: "Patron",
                            foreign_key: "patron_id"
    end
```
________________________

### **10.touch**
________________________

```ruby
    class Order < ActiveRecord::Base
      belongs_to :customer, touch: true
    end
     
    class Customer < ActiveRecord::Base
      has_many :orders
    end
```
order中的任何一个对象发生save或者destroy的时候就会更新拥有这个order的customer的updated_at,并将updated_at设置为当前时间

________________________




curl -XPUT http://localhost:9200/medcl/ -d'
{
    "index" : {
        "analysis" : {
            "analyzer" : {
                "pinyin_analyzer" : {
                    "tokenizer" : "my_pinyin",
                    "filter" : ["standard"]
                }
            },
            "tokenizer" : {
                "my_pinyin" : {
                    "type" : "pinyin",
                    "first_letter" : "prifix",
                    "padding_char" : ""
                }
            }
        }
    }
}'