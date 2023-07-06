---
layout: post
title: "PostgreSQL Full Text Search with Java and Spring Boot"
date: 2022-03-20 18:34:23 -0600
categories: Java, Spring Boot, PostgreSQL
excerpt_separator: <!--more-->
---

There are a lot of blog posts showing how PostgreSQL full-text search features work however in this post, I wanted to share an approach to how this feature would work with a hypothetical e-commerce store for searching products according to the product name and their attributes.

<!--more-->

Whenever I wanted to implement a search feature for my projects, I usually used SQL-like operators to concatenate the input with % to match any number of characters or \_ to match a single letter as a prefix or suffix. This approach can have potential security issues like SQL injection, lead to performance issues while handling a large number of concurrent connections, and the results will lead to inaccurate results due to misspelled or incomplete words in the search term.

## Full-Text Search in PostgreSQL

For doing full-text searches in PostgreSQL, the database has enabled by default the ability to use two specific functions to_tsquery and to_tsvector, which makes life easier when searching inside the content of a text column. The former enables us to create, using text-search-specific operators, complex queries the latter allows us to create tokenized columns from the text content we are storing in the database.

For this example, we will create two columns, the first for storing the information for viewing and the second for searching using the to_tsvector function. In the second part of the article, we will create a search wrapper function using the to_tsquery function for Hibernate Framework.

## Database Schema

First lets us design the database schema with something like the image below.

![Products DB](/assets/product-search-db.webp)

## Products Schema

We define an ecommerce_products table which will be our main entity and will store all related information of products, then we define an ecommerce_attributes table whose information retains the characteristics of the products, and finally, categories where each category can have more than one product and one product can have more than one category for many-to-many relationship.

Now that we have our model in place we can start creating our Hibernate entities.

```java
@Table(name = "ecommerce_categories")
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "category_seq")
    @SequenceGenerator(name = "category_seq", sequenceName = "ecommerce_categories_id_seq", allocationSize = 1)
    public Long id;

    public String name;

    public String description;

    // getters & setters omitted
}
```

Nothing special we created our entity with an id generated from a sequence.

```java
@Table(name = "ecommerce_attributes")
public class ProductAttribute {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_attribute_id_seq")
    @SequenceGenerator(name = "product_attribute_id_seq", sequenceName = "ecommerce_attributes_id_seq", allocationSize = 1)
    private Long id;

    private String attribute;

    private String value;

    @JsonIgnore
    @Column(name = "attribute_value_ts", updatable = false, insertable = false)
    private String attributeSearch;

    @JsonIgnore
    @ManyToOne
    private Product product;

    // getters & setters omitted
}
```

Note how we define attributeSearch field as non-updatable and non-insertable and marked with @JsonIgnore

```java
@Table(name = "ecommerce_products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_id_seq")
    @SequenceGenerator(name = "product_id_seq", sequenceName = "ecommerce_products_id_seq", allocationSize = 1)
    private Long id;

    private String serialCode;

    private String name;

    private String description;

    private BigDecimal price;

    @JsonIgnore
    @Column(name = "name_ts", updatable = false, insertable = false)
    private String nameSearch;

    @OneToMany(mappedBy = "product", cascade = CascadeType.PERSIST)
    private Set<ProductAttribute> attributes;

    @ManyToMany
    @JoinTable(name = "ecommerce_products_join_categories", joinColumns = {
            @JoinColumn(name = "product_id", nullable = false, updatable = false)},
            inverseJoinColumns = {@JoinColumn(name = "category_id", nullable = false, updatable = false)})
    @Cascade({
            org.hibernate.annotations.CascadeType.MERGE
    })
    private Set<Category> categories = new HashSet<>();

}
```

Our main entity with collections @OneToMany and @ManyToMany for creating relationships with the entities Category and ProductAttribute

## Database Inserts

Once we have defined our model we need to populate the database which we will do it through a trigger.

```sql
create or replace function update_products_ts_fields()
returns trigger
language plpgsql
as

$$
begin
    update ecommerce_products
    set name_ts = to_tsvector(name)
    where id = new.id;

    RETURN new;
end;
$$;
```

First we create a function which will be later used in the trigger

```sql
create trigger trg_update_products_ts_field
    after insert
    on ecommerce_products
    for each row
execute function update_products_ts_fields();
```

Whenever we try to insert a row in the table will update the row with the to_tsvector function.

Hibernate Function and SQL Dialect

For creating a custom function in HQL (Hibernate Query Language) we will need to create a class implementing the interface SQLFunction like in the next example.

```java
public class PostgreSQLSearchFunction implements SQLFunction {

    private StringBuilder fragment = new StringBuilder();

    @Override
    public boolean hasArguments() {
        return true;
    }

    @Override
    public boolean hasParenthesesIfNoArguments() {
        return false;
    }

    @Override
    public Type getReturnType(Type firstArgumentType, Mapping mapping) throws QueryException {
        return new BooleanType();
    }

    @Override
    public String render(Type firstArgumentType, List arguments, SessionFactoryImplementor factory) throws QueryException {
        if (arguments == null || arguments.size() < 2) {
            throw new IllegalArgumentException("The function must be passed at least 2 arguments");
        }

        fragment.setLength(0);
        String ftsConfiguration = null;
        String field = null;
        String value = null;

        field = (String) arguments.get(0);
        value = (String) arguments.get(1);

        fragment.append(field);
        fragment.append(" @@ ");
        fragment.append("to_tsquery(");
        fragment.append(value);
        fragment.append(")");

        return fragment.toString();
    }
}
```

And for making available the search function in HQL we define a custom dialect.

```java
public class ProductPostgreSQLDialect extends PostgreSQL10Dialect {

    public final static String SEARCH_FUNCTION = "search";

    public ProductPostgreSQLDialect() {
        registerFunction(SEARCH_FUNCTION, new PostgreSQLSearchFunction());
    }
}
```

In the custom dialect, we use the registerFunction with the name of the new custom function and an instance of the class PostgreSQLSearchFunction.

Querying our model with HQL

Now that we have everything set up in place we can query our entities with the following sentence.

```sql
select p
from Product p
left join Attributes att
left join Categories cat
where product.id in (
select p1.id
from Product p1
left join Attributes att1
left join Categories cat1
where search(p1.nameSearch, ?) and search(att1.attributeSearch, ?) and cat1.id = ?
)
```

The query is right straightforward, in the section of the subquery, in the where clause, we can see we are using the search function the first parameter we pass is the column we are searching, and the second parameter we pass is the actual query or what we are looking for.

## Conclusion

Doing a full-text search in java/spring-boot is quite expensive in terms of lines of code compared with other languages however with this approach I could achieve a really fast execution longing 200ms for the whole request-response call and also remember that with this approach I can abstract the database implementation and change it for another let’s say Oracle or MS SQL Server by just changing the Dialect and the SQLFunction, no need to change all the queries in advance
