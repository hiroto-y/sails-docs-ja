# Associations
With Sails and Waterline, you can associate models across multiple data stores. This means that even if your users live in PostgreSQL and their photos live in MongoDB, you can interact with the data as if they lived together in the same database. You can also have associations that span different connections (i.e. datastores/databases) using the same adapter. This comes in handy if, for example, your app needs to access/update legacy recipe data stored in a MySQL database in your company's data center, but also store/retrieve ingredient data from a brand new MySQL database in the cloud.

##Dominance
###Example Ontology

```
// User.js
module.exports = {
  connection: 'ourMySQL',
  attributes: {
    email: 'string',
    wishlist: {
      collection: 'product',
      via: 'wishlistedBy'
    }
  }
};
```

```
// Product.js
module.exports = {
  connection: 'ourRedis',
  attributes: {
    name: 'string',
    wishlistedBy: {
      collection: 'user',
      via: 'wishlist'
    }
  }
};
```

### The Problem

It's easy to see what's going on in this cross-adapter relationship. There's a many-to-many ( N->... ) relationship between users and products. In fact, you can imagine a few other relationships (e.g. purchases) which might exist, but since those are probably better-represented using a middleman model, I went for something simple in this example.

Anyways, that's all great... but where does the relationship resource live? "ProductUser", if you'll pardon with the SQL-oriented nomenclature. We know it'll end up on one side or the other, but what if we want to control which database it ends up in?

```
IMPORTANT NOTE
This is only a problem because both sides of the association have a via modifier specified!! In the absence of via, a collection attribute always behaves as dominant: true. See the FAQ below for more information.
```

###The Solution

Eventually, it may be even be possible to specify a 3rd connection/adapter to use for the join table. For now, we'll focus on choosing one side or the other.

We address this through the concept of "dominance." In any cross-adapter model relationship, one side is assumed to be dominant. It may be helpful to think about the analogy of a child with multinational parents who must choose one country or the other for her citizenship

Here's the ontology again, but this time we'll indicate the MySQL database as the "dominant". This means that the "ProductUser" relationship "table" will be stored as a MySQL table.

```
// User.js
module.exports = {
  connection: 'ourMySQL',
  attributes: {
    email: 'string',
    wishlist: {
      collection: 'product',
      via: 'wishlistedBy',
      dominant: true
    }
  }
};
```

```
// Product.js
module.exports = {
  connection: 'ourRedis',
  attributes: {
    name: 'string',
    wishlistedBy: {
      collection: 'user',
      via: 'wishlist'
    }
  }
};
```

###Choosing a "dominant"

Several factors may influence your decision:

* If one side is a SQL database, placing the relationship table on that side will allow your queries to be more efficient, since the relationship table can be joined before the other side is communicated with. This reduces the number of total queries required from 3 to 2.
* If one connection is much faster than the other, all other things being equal, it probably makes sense to put the connection on that side.
* If you know that it is much easier to migrate one of the connections, you may choose to set that side as dominant. Similarly, regulations or compliance issues may affect your decision as well. If the relationship contains sensitive patient information (for instance, a relationship between Patient and Medicine) you want to be sure that all relevant data is saved in one particular database over the other (in this case, Patient is likely to be dominant).
* Along the same lines, if one of your connections is read-only (perhaps Medicine in the previous example is connected to a read-only vendor database), you won't be able to write to it, so you'll want to make sure your relationship data can be persisted safely on the other side.

###FAQ

####What if one of the collections doesn't have via?

If a collection association does not have a via property, it is automatically dominant: true.

####What if both collections don't have via?

If both collections don't have via, then they are not related. Both are dominant, because they are separate relationship tables!!

####What about model associations?

In all other types of associations, the dominant property is prohibited. Setting one side to dominant is only necessary for associations between two models which have an attribute like: { via: '...', collection: '...' } on both sides.

####Can a model be dominant for one attribute and not another?

Keep in mind that a model is "dominant" only in the context of a particular relationship. A model may be dominant in one or more relationships (attributes) while simultaneously NOT being dominant in other relationships (attributes). e.g. if a User has a collection of toys called favoriteToys via favoriteToyOf on the Toy model, and favoriteToys on User is dominant: true, Toy can still be dominant in other ways. So Toy might also be associated to User by way of its attribute, designedBy, for which it is dominant: true.

####Can both models be dominant?

No. If both models in a cross-adapter/cross-connection, many-to-many association set dominant: true, an error is thrown before lift.

####Can neither model be dominant?

Sort of... If neither model in a cross-adapter/cross-connection, many-to-many association sets dominant: true, a warning is displayed before lift, and a guess will be made automatically based on the characteristics of the relationship. For now, that just means an arbitrary decision based on alphabetical order :)

####What about non-cross-adapter associations?

The dominant property is silently ignored in non-cross-adapter/cross-connection associations. We're assuming you might be planning on breaking up the schema across multiple connections eventually, and there's no reason to prevent you from being proactive. Plus, this reserves additional future utility for the "dominant" option down the road.

##Many-to-Many

###Overview

A many-to-many association states that a model can be associated with many other models and vice-versa. Because both models can have many related models a new join table will need to be created to keep track of these relations.
Waterline will look at your models and if it finds that two models both have collection attributes that point to each other, it will automatically build up a join table for you.
Because you may want a model to have multiple many-to-many associations on another model a via key is needed on the collection attribute. This states which model attribute on the one side of the association is used to populate the records.
Using the User and Pet example lets look at how to build a schema where a User may have many Pet records and a Pet may have multiple owners.

###Many-to-Many Example

In this example, we will start with an array of users and an array of pets. We will create records for each element in each array then associate all of the Pets with all of the Users. If everything worked properly, we should be able to query any User and see that they 'own' all of the Pets. Furthermore, we should be able to query any Pet and see that it is 'owned' by every User.

myApp/api/models/pet.js

```
module.exports = {

    attributes: {
        name:'STRING',
        color:'STRING',

        // Add a reference to User
        owners: {
            collection: 'user',
            via: 'pets',
            dominant:true
        }
    }
}
```
myApp/api/models/user.js

```
module.exports = {

    attributes: {
        name:'STRING',
        age:'INTEGER',

        // Add a reference to Pet
        pets:{
            collection: 'pet',
            via: 'owners'
        }
    }

}
```

myApp/config/bootstrap.js

```
module.exports.bootstrap = function (cb) {

// After we create our users, we will store them here to associate with our pets
var storeUsers = []; 

var users = [{name:'Mike',age:'16'},{name:'Cody',age:'25'},{name:'Gabe',age:'107'}];
var ponys = [{ name: 'Pinkie Pie', color: 'pink'},{ name: 'Rainbow Dash',color: 'blue'},{ name: 'Applejack', color: 'orange'}]

// This does the actual associating.
// It takes one Pet then iterates through the array of newly created Users, adding each one to it's join table
var associate = function(onePony,cb){
  var thisPony = onePony;
  var callback = cb;

  storeUsers.forEach(function(thisUser,index){
    console.log('Associating ',thisPony.name,'with',thisUser.name);
    thisUser.pets.add(thisPony.id);
    thisUser.save(console.log);

    if (index === storeUsers.length-1)
      return callback(thisPony.name);
  })
};


// This callback is run after all of the Pets are created.
// It sends each new pet to 'associate' with our Users  
var afterPony = function(err,newPonys){
  while (newPonys.length){
    var thisPony = newPonys.pop();
    var callback = function(ponyID){
      console.log('Done with pony ',ponyID)
    }
    associate(thisPony,callback)
  }
  console.log('Everyone belongs to everyone!! Exiting.');

  // This callback lets us leave bootstrap.js and continue lifting our app!
  return cb()
};

// This callback is run after all of our Users are created.
// It takes the returned User and stores it in our storeUsers array for later.
var afterUser = function(err,newUsers){
  while (newUsers.length)
    storeUsers.push(newUsers.pop())

  Pet.create(ponys).exec(afterPony)
};


User.create(users).exec(afterUser)

};
```

Lifting our app with sails console

```
dude@littleDude:~/node/myApp$ sails console

info: Starting app in interactive mode...

Associating  Applejack with Gabe
Associating  Applejack with Cody
Associating  Applejack with Mike
Done with pony  Applejack
Associating  Rainbow Dash with Gabe
Associating  Rainbow Dash with Cody
Associating  Rainbow Dash with Mike
Done with pony  Rainbow Dash
Associating  Pinkie Pie with Gabe
Associating  Pinkie Pie with Cody
Associating  Pinkie Pie with Mike
Done with pony  Pinkie Pie
Everyone belongs to everyone!! Exiting.
info: Welcome to the Sails console.
info: ( to exit, type <CTRL>+<C> )

sails> null { name: 'Gabe',
  age: 107,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 9 }
null { name: 'Cody',
  age: 25,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 8 }
null { name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
null { name: 'Gabe',
  age: 107,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 9 }
null { name: 'Cody',
  age: 25,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 8 }
null { name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
null { name: 'Gabe',
  age: 107,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 9 }
null { name: 'Cody',
  age: 25,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 8 }
null { name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
sails> Pet.find().populate('owners').exec(function(e,r){while(r.length){var thisPet=r.pop();console.log(thisPet.toJSON())}});
{ owners: 
   [ { name: 'Mike',
       age: 16,
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Cody',
       age: 25,
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Gabe',
       age: 107,
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Applejack',
  color: 'orange',
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 9 }
{ owners: 
   [ { name: 'Mike',
       age: 16,
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Cody',
       age: 25,
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Gabe',
       age: 107,
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Rainbow Dash',
  color: 'blue',
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 8 }
{ owners: 
   [ { name: 'Mike',
       age: 16,
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Cody',
       age: 25,
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Gabe',
       age: 107,
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Pinkie Pie',
  color: 'pink',
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
sails> User.find().populate('pets').exec(function(e,r){while(r.length){var thisUser=r.pop();console.log(thisUser.toJSON())}});
{ pets: 
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Rainbow Dash',
       color: 'blue',
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Applejack',
       color: 'orange',
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Gabe',
  age: 107,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 9 }
{ pets: 
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Rainbow Dash',
       color: 'blue',
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Applejack',
       color: 'orange',
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Cody',
  age: 25,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 8 }
{ pets: 
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 7,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Rainbow Dash',
       color: 'blue',
       id: 8,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) },
     { name: 'Applejack',
       color: 'orange',
       id: 9,
       createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
       updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST) } ],
  name: 'Mike',
  age: 16,
  createdAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  updatedAt: Wed Feb 12 2014 18:06:50 GMT-0600 (CST),
  id: 7 }
```

###Notes

For a more detailed description of this type of association, see the Waterline Docs

##One Way Association
###Overview

A one way association is where a model is associated with another model. You could query that model and populate to get the associatED model. You can't however query the associated model and populate to get the associatING model.

###One Way Example

In this example, we are associating a User with a Pet but not a Pet with a User.

myApp/api/models/pet.js

```
module.exports = {

    attributes: {
        name:'STRING',
        color:'STRING'
    }

}
```

myApp/api/models/user.js

```
module.exports = {

    attributes: {
        name:'STRING',
        age:'INTEGER',
        pony:{
            model: 'pet'
        }
    }

}
```

Using sails console

```
sails> Pet.create({name:'Pinkie Pie',color:'pink'}).exec(console.log)
null { name: 'Pinkie Pie',
  color: 'pink',
  createdAt: Tue Feb 11 2014 15:45:33 GMT-0600 (CST),
  updatedAt: Tue Feb 11 2014 15:45:33 GMT-0600 (CST),
  id: 5 }

sails> User.create({name:'Mike',age:21,pony:5}).exec(console.log);
null { name: 'Mike',
  age: 21,
  pony: 5,
  createdAt: Tue Feb 11 2014 15:48:53 GMT-0600 (CST),
  updatedAt: Tue Feb 11 2014 15:48:53 GMT-0600 (CST),
  id: 1 }

sails> User.find({name:'Mike'}).populate('pony').exec(console.log);
null [ { name: 'Mike',
    age: 21,
    pony: 
     { name: 'Pinkie Pie',
       color: 'pink',
       id: 5,
       createdAt: Tue Feb 11 2014 15:45:33 GMT-0600 (CST),
       updatedAt: Tue Feb 11 2014 15:45:33 GMT-0600 (CST) },
    createdAt: Tue Feb 11 2014 15:48:53 GMT-0600 (CST),
    updatedAt: Tue Feb 11 2014 15:48:53 GMT-0600 (CST),
    id: 1 } ]
```

###Notes

For a more detailed description of this type of association, see the Waterline Docs
Because we have only formed an association on one of the models, a Pet has no restrictions on the number of User models it can belong to. If we wanted to, we could change this and associate the Pet with exactly one User and the User with exactly one Pet.

##One-to-Many
###Overview

A one-to-many association states that a model can be associated with many other models. To build this association a virtual attribute is added to a model using the collection property. In a one-to-many association one side must have a collection attribute and the other side must contain a model attribute. This allows the many side to know which records it needs to get when a populate is used.
Because you may want a model to have multiple one-to-many associations on another model a via key is needed on the collection attribute. This states which model attribute on the one side of the association is used to populate the records.

###One-to-Many Example

myApp/api/models/pet.js

```
module.exports = {

    attributes: {
        name:'STRING',
        color:'STRING',
        owner:{
            model:'user'
        }
    }

}
```

myApp/api/models/user.js

```
module.exports = {

    attributes: {
        name:'STRING',
        age:'INTEGER',
        pets:{
            collection: 'pet',
            via: 'owner'
        }
    }

}
```

Using sails console

```
sails> User.create({name:'Mike',age:'21'}).exec(console.log)
null { pets: [Getter/Setter],
  name: 'Mike',
  age: 21,
  createdAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST),
  updatedAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST),
  id: 1 }

sails> Pet.create({name:'Pinkie Pie',color:'pink',owner:1}).exec(console.log)
null { name: 'Pinkie Pie',
    color: 'pink',
    owner: 1,
    createdAt: Tue Feb 11 2014 17:58:04 GMT-0600 (CST),
    updatedAt: Tue Feb 11 2014 17:58:04 GMT-0600 (CST),
    id: 2 }

sails> Pet.create({name:'Applejack',color:'orange',owner:1}).exec(console.log)
null { name: 'Applejack',
    color: 'orange',
    owner: 1,
    createdAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
    updatedAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
    id: 4 }

sails> User.find().populate('pets').exec(function(err,r){console.log(r[0].toJSON())});
{ pets: 
   [ { name: 'Pinkie Pie',
       color: 'pink',
       id: 2,
       createdAt: Tue Feb 11 2014 17:58:04 GMT-0600 (CST),
       updatedAt: Tue Feb 11 2014 17:58:04 GMT-0600 (CST),
       owner: 1 },
     { name: 'Applejack',
       color: 'orange',
       id: 4,
       createdAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
       updatedAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
       owner: 1 } ],
  name: 'Mike',
  age: 21,
  createdAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST),
  updatedAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST),
  id: 1 }

sails> Pet.find(4).populate('owner').exec(console.log)
null [ { name: 'Applejack',
    color: 'orange',
    owner: 
     { pets: [Getter/Setter],
       name: 'Mike',
       age: 21,
       id: 1,
       createdAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST),
       updatedAt: Tue Feb 11 2014 17:49:04 GMT-0600 (CST) },
    createdAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
    updatedAt: Tue Feb 11 2014 18:02:58 GMT-0600 (CST),
    id: 4 } ]
```

###Notes

For a more detailed description of this type of association, see the Waterline Docs

##One-to-One
###Overview

A one-to-one association states that a model may only be associated with one other model. In order for the model to know which other model it is associated with, a foreign key must be included in the record.

###One-to-One Example

In this example, we are associating a Pet with a User. The User may only have one Pet in this case but a Pet is not limited to a single User.

myApp/api/models/pet.js

```
module.exports = {

    attributes: {
        name:'STRING',
        color:'STRING',
        owner:{
            model:'user'
        }
    }

}
```

myApp/api/models/user.js

```
module.exports = {

    attributes: {
        name:'STRING',
        age:'INTEGER',
        pony:{
            model: 'pet'
        }
    }

}
```

Using sails console

```
sails> User.create({ name: 'Mike', age: 21}).exec(console.log);
null { name: 'Mike',
  age: 21,
  createdAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST),
  updatedAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST),
  id: 1 }

sails> Pet.create({ name: 'Pinkie Pie', color: 'pink', owner: 1}).exec(console.log)
null { name: 'Pinkie Pie',
    color: 'pink',
    owner: 1,
    createdAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
    updatedAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
    id: 2 }

sails> Pet.find().populate('owner').exec(console.log)
null [ { name: 'Pinkie Pie',
    color: 'pink',
    owner: 
     { name: 'Mike',
       age: 21,
       id: 1,
       createdAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST),
       updatedAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST) },
    createdAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
    updatedAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
    id: 2 } ]

sails> User.find().populate('pony').exec(console.log)
null [ { name: 'Mike',
    age: 21,
    createdAt: Thu Feb 20 2014 18:11:15 GMT-0600 (CST),
    updatedAt: Thu Feb 20 2014 18:11:15 GMT-0600 (CST),
    id: 2,
    pony: undefined } ]

sails> User.update({name:'Mike'},{pony:2}).exec(console.log)
null [ { name: 'Mike',
    age: 21,
    createdAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST),
    updatedAt: Thu Feb 20 2014 17:30:58 GMT-0600 (CST),
    id: 1,
    pony: 2 } ]

sails> User.findOne(1).populate('pony').exec(console.log)
null { name: 'Mike',
  age: 21,
  createdAt: Thu Feb 20 2014 17:12:18 GMT-0600 (CST),
  updatedAt: Thu Feb 20 2014 17:30:58 GMT-0600 (CST),
  id: 1,
  pony: 
   { name: 'Pinkie Pie',
     color: 'pink',
     id: 2,
     createdAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
     updatedAt: Thu Feb 20 2014 17:26:16 GMT-0600 (CST),
     owner: 1 } }
```

###Notes

For a more detailed description of this type of association, see the Waterline Docs

## Through Associations
###Overview

Many-to-Many through associations behave the same way as many-to-many associations with the exception of the join table being automatically created for you. This allows you to attach additional attributes onto the relationship inside of the join table.
Unfortunately, they are not supported yet. Don't worry though, there's an easy workaround.
You can accomplish this by using an additional model as an intermediary. Instead of a many-to-many association between two models, you can use multiple one-to-many associations through the intermediary model.

##Direct Access to Join Tables
###DANGER DANGER!!

You can use waterline for low level access to join tables. Unless you know what youre doing, DONT DO THIS!

TODO