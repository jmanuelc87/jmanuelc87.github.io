---
layout: post
title: "Hibernate and Spring Boot, the N+1 query problem"
date: 2022-03-27 18:34:23 -0600
categories: Java, Spring Boot, PostgreSQL
excerpt_separator: <!--more-->
---
The N+1 problem is usually associated with the usage of Hibernate & JPA implementations and causes performance degradation and bottleneck situations however there's a known feature for solving the N+1 problem the Entity Graphs.

<!--more-->

## What Entity Graphs are trying to solve?

When using HQL (Hibernate Query Language) for retrieving entity instances the HQL query gets translated to actual SQL however when the currently selected entity has relations many-to-one or many-to-many the Hibernate engine also creates SQL sentences for each of the related entities just for filling up fields, therefore, the database ends processing N queries for related entities and 1 query for the currently selected entity.

Entity Graphs solve the problem by defining some metadata information in the entity and turning the relations to type lazy for the Hibernate Engine to figure out how to populate the fields in the entity by sending the query once.

Hands on, where to start?

First of all, let's bring up the entities from this post PostgreSQL Full Text Search with Java and Spring Boot where we have three entities Product and ProductAttribute & Category where the former is the main entity class and the latter are classes depending on the main entity.

For one-to-many, many-to-many relationships we must add the next property as part of the attributes to the Hibernate annotations @OneToMany  @ManyToMany let's look at the code below.

```java
@OneToMany(fetch = FetchType.LAZY)

@ManyToMany(fetch = FetchType.LAZY)
```

The next step is to provide the Entity Graph an annotation at the top of the main entity class in this case is Product a class like the code below.

```java
@NamedEntityGraph(name = "Product.searchProducts",
        attributeNodes = {@NamedAttributeNode("attributes"), @NamedAttributeNode("categories")}) //3
public class Product {

    // ...

    @OneToMany(mappedBy = "product", cascade = CascadeType.PERSIST, fetch = FetchType.LAZY) //1
    private Set<ProductAttribute> attributes;

    @ManyToMany(fetch = FetchType.LAZY) //2
    @JoinTable(name = "ecommerce_products_join_categories", joinColumns = {
            @JoinColumn(name = "product_id", nullable = false, updatable = false)},
            inverseJoinColumns = {@JoinColumn(name = "category_id", nullable = false, updatable = false)})
    @Cascade({
            org.hibernate.annotations.CascadeType.MERGE
    })
    private Set<Category> categories = new HashSet<>();
}
```

The relevant code is at the beginning of the snippet, the annotation @NamedEntityGraph with the attribute name will use later for making reference to, and attributesNodes will accept an array of @NamedAttributesNode making reference to the fields of the current class (attributes and categories) that will need to retrieve data from the rows with a single query.

## Entity Retrieval

For retrieving our entities we need to pass a hint to the EntityManager specifying which @NamedEntityGraph we want to use. Let's see the code below.

```java
public class ProductRepositoryCustomImpl {

    private EntityManager em;

    // Constructor omitted for clarity

    private List<Product> searchProducts(...) {
        // create a query with parameters
        EntityGraph<?> eg = em.getEntityGraph("product-entity-graph");
        return em.createQuery(...)
                .setHint("javax.persistence.fetchgraph", eg)
    }
}
```

And if you're using spring data jpa you could use an annotation @EntityGraph with the value of the name entity graph, you want to use.

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    
     @EntityGraph(value = "Product.searchProducts")
     List<Product> searchProducts(...);
}
```

## Conclusion

Thats it! easy solution for avoiding a recurrent problem using hibernate for avoiding the framework using more SQL statements than necessary.