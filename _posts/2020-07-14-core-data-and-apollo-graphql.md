---
layout: post
title:  Core Data and Apollo GraphQL
date:   2020-07-14 18:02:40 -0600
tags:   [coredata apollo graphql]
---
By employing a protocol-based approach to Core Data and implementing fragments in the [Apollo iOS client](https://github.com/apollographql/apollo-ios), robust caching becomes a trivial process.

## CoreData
Even with some recent improvements, Core Data has never been easy to use. I use a protocol-based approach to simplify repetitive tasks and make Core Data more extensible.

### ManagedObject/ManagedObjectSupport
I start by defining a class named ManagedObject, a subclass of NSManagedObject, that will be the superclass of all other Core Data objects. At a minimum, this class contains an idString property that simplifies creation and retrieval of objects. ManagedObject is also a good place to define any properties and methods that should be common to all objects.

```swift
class ManagedObject: NSManagedObject {
  @NSManaged var idString: String
}
```

Additionally, I define a protocol named ManagedObjectSupport. This protocol has no requirements, but allows me to implement static methods for object creation/retrieval and fetched results controller construction.

```swift
protocol ManagedObjectSupport {}
extension ManagedObject: ManagedObjectSupport {}

extension ManagedObjectSupport where Self: ManagedObject {
  // Creates or retrieves an object with the specified id
  static func object(in context: NSManagedObjectContext, withId id: String) -> Self {
    let request = Self.fetchRequest() as! NSFetchRequest<Self>
    request.predicate = NSPredicate(format: "%K == %@",  #keyPath(idString), id)
    request.fetchLimit = 1
    request.returnsObjectsAsFaults = false

    switch (try? context.fetch(request))?.first {
    case .some(let object):
      return object

    case .none:
      let newObject = Self(context: context)
      newObject.idString = id
      return newObject
    }
  }

  // Returns an existing object with the specified id
  static func existingObject(in context: NSManagedObjectContext, withId id: String) -> Self? {
    let request = Self.fetchRequest() as! NSFetchRequest<Self>
    request.predicate = NSPredicate(format: "%K == %@", #keyPath(idString), id)
    request.fetchLimit = 1
    request.returnsObjectsAsFaults = false
    return (try? context.fetch(request))?.first
  }

  // Configure and return a fetched results controller
  static func fetchedResultsController(in context: NSManagedObjectContext,
                                       sortDescriptors: [NSSortDescriptor]? = nil,
                                       predicate: NSPredicate? = nil,
                                       sectionKeyPath: String? = nil) -> NSFetchedResultsController<Self> {
    let request = Self.fetchRequest() as! NSFetchRequest<Self>
    request.sortDescriptors = sortDescriptors
    request.predicate = predicate
    return NSFetchedResultsController(fetchRequest: request, managedObjectContext: context,
                                      sectionNameKeyPath: sectionKeyPath, cacheName: nil)
  }
```

I can now define and access a Person subclass using the methods in ManagedObjectSupport.

```swift
class Person: ManagedObject {
  // idString is inherited from the ManagedObject superclass
  
  @NSManaged var firstName: String?
  @NSManaged var lastName: String?
  @NSManaged var nickname: String?
}

let p = Person.object(in: <someContext>, withId: "abc")
let e = Person.existingObject(in: <someContext>, withId: "abc")
let f = Person.fetchedResultsController(in: <someContext>, sortDescriptors: <someDescriptors>)
```

## Apollo GraphQL
When researching GraphQL options for iOS, Apollo’s client quickly rose to the top of my list. It is open-source, actively supported, and written entirely in Swift. The client contains tooling to generate type-safe Swift code from a GraphQL schema, but that code can sometimes be complicated to use. The use of GraphQL fragments has simplified the generated code and led to easy reuse of types.

### GraphQL Fragments
Normally, we declare the returned data from GraphQL operations with individual fields from a type defined in the schema.

```graphql
query Person($id: ID!) {
  person(id: $id) {
    id
    firstName
    lastName
    nickname
  }
}

query AllPersons() {
  allPersons {
    id
    firstName
    lastName
    nickname
  }
}
```
However, Apollo’s code-generation will scope the return fields to each operation, creating duplication and making it difficult to reuse these repeated structures. Fortunately, GraphQL fragments are the perfect solution to this problem (read more about fragments [here](https://graphql.org/learn/queries/#fragments)). In nearly every case, I start by defining a fragment for the GraphQL type and use that fragment in any operation returns. Unifying the types in this way makes for greater ease of use and also reduces the size of Apollo’s generated code.

```graphql
fragment PersonFragment on Person {
  id
  firstName
  lastName
  nickname
}

query Person($id: ID!) {
  person(id: $id) {
    ...PersonFragment
  }
}

query AllPersons() {
  allPersons {
    ...PersonFragment
  }
}
```

## Putting it together
On their own, a protocol-based approach to Core Data and the use of GraphQL fragments are valuable tools. When used together, though, object creation from returned GraphQL data becomes trivial while retaining type safety.

### FragmentUpdatable
I define a protocol named FragmentUpdatable, which ManagedObject subclasses use to specify what type of fragment to receive data from. This protocol is then extended to leverage the ManagedObjectSupport protocol for object creation and retrieval.

```swift
protocol FragmentUpdatable {
  associatedtype Fragment: GraphQLFragment & Identifiable
  func update(with fragment: Fragment)
}

extension FragmentUpdatable where Self: ManagedObject {
  static func object(in context: NSManagedObjectContext, withFragment fragment: Self.Fragment?) -> Self? {
    guard let fragment = fragment, let id = fragment.id as? String else { return nil }
    let object = Self.object(in: context, withId: id)
    object.update(with: fragment)
    return o
  }
}
```

Each ManagedObject subclass can conform to FragmentUpdatable to describe which fragment it accepts and how to map the data. Notice that the protocol requires the Fragment type to be Identifiable. Since PersonFragment already has an id property, an empty extension is all that is necessary.

```swift
extension PersonFragment: Identifiable {}

extension Person: FragmentUpdatable {
  typealias Fragment = PersonFragment

  func update(with fragment: Fragment) {
    self.firstName = fragment.firstName
    self.lastName = fragment.lastName
    self.nickname = fragment.nickname
  }
}
```

Now, after a GraphQL operation returns, I can create/retrieve/update in one line.

```swift
let person = Person.object(in: <someContext>, withFragment: <somePersonFragment>)
```




