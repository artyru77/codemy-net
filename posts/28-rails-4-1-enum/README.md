One of the coolest thing introduced in Rails 4.1 is the Enum. 

We often need to store the state of an object in the state itself. We even use these states to create state machines for our objects. Rails Enum is a great way for storing our object's state.

## Notable Links

+ [Pull Request Illustrating Enum](https://github.com/codemy/blogmenow/pull/3/files) 


## What is an object's state?

Some types of objects in your application will need to have some sort of state. For example

+ Posts can be in 'draft' or 'published' mode
+ Reservations can be 'created', 'confirmed' or 'cancelled'
+ Bank Accounts can be 'opened', 'frozen' or 'deactivated'

Each state in an object represents something. The state usually tells the user how to act towards that object. 

For example in a hotel a reservation that is 'created' may just mean the user is browsing through the booking engine and haven't decided to book yet. A 'confirmed' reservation means that the customer has booked the room and put down the deposit. A 'cancelled' reservation tells the hotel staff that a particular reservation has been cancelled.

Usually when working with states there are many gems that help us in our rails app, gems like [state_machine](https://github.com/pluginaweek/state_machine) or [AASM](https://github.com/aasm/aasm) 

The new Enum feature in rails replaces a lot of the features provided by the above gems. Enum also serves as a great foundation for building more complex state machines like transitions, and guards and more.