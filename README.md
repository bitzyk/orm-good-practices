# orm-good-practices
good practices compilation based on PHP UK Conference 2016 - Marco Pivetta - Doctrine ORM Good Practices and Tricks (https://www.youtube.com/watch?v=rzGeNYC3oz0)

#### Principles


* Entities should work without ORM
* Entities should work without the DB
* Entities are not type arrays:
 ```php
<?php

class User
{
    /**
     * @var string
     */
    private $username;
    
    private $password;

    /**
     * @return string
     */
    public function getUsername(): string
    {
        return $this->username;
    }

    /**
     * @param string $username
     * @return User
     */
    public function setUsername(string $username): User
    {
        $this->username = $username;
        return $this;
    }

    /**
     * @return mixed
     */
    public function getPassword()
    {
        return $this->password;
    }

    /**
     * @param mixed $password
     * @return User
     */
    public function setPassword($password)
    {
        $this->password = $password;
        return $this;
    }
    
}
```

* Entities have a behaviour - you should think on how objects talk (interract) to each other instead of thinking on what object contains
e.g. instead of asking what does the user contains, you can ask does the user can tell me his username

```php
<?php

class User
{
    /**
     * @var string
     */
    private $username;

    /**
     * @var bool
     */
    private $banned;
    
    public function toNickname() : string 
    {
        return $this->username;
    }
    
    public function authenticate(
        string $pass, AuthServiceInterface $authService
    ) : bool 
    {
        return $authService->authenticate($this->username, $pass)
            && ! $this->hasActiveBans();
    }

}
```

* respect the law of demeter
    * The Law of Demeter (LoD) or principle of least knowledge is a design guideline for developing software, particularly object-oriented programs. In its general form, the LoD is a specific case of loose coupling.
    * Each unit should have only limited knowledge about other units: only units "closely" related to the current unit.
    * Each unit should only talk to its friends; don't talk to strangers.
    * Only talk to your immediate friends. 
    * When applied to object-oriented programs, the Law of Demeter can be more precisely called the “Law of Demeter for Functions/Methods” (LoD-F). In this case, an object A can request a service (call a method) of an object instance B, but object A should not "reach through" object B to access yet another object, C, to request its services. Doing so would mean that object A implicitly requires greater knowledge of object B's internal structure. Instead, B's interface should be modified if necessary so it can directly serve object A's request, propagating it to any relevant subcomponents. Alternatively, A might have a direct reference to object C and make the request directly to that. If the law is followed, only object B knows its own internal structure.
    * The advantage of following the Law of Demeter is that the resulting software tends to be more maintainable and adaptable. Since objects are less dependent on the internal structure of other objects, object containers can be changed without reworking their callers.
    
* deduction -> Dissalow collection access from outside the entity (keep collection hidden in your entities)
    * Advantages
        * More expressive
        * Easier to test
        * Less coupling
        * More flexible
        * Easier to refactor
        
 ```php
 <?php

class User
{
    public function hasAccessTo(Resource $resource) : bool 
    {
        return (bool) array_filter(
            $this->role->getAccessLevels(),
            function(AccessLevel $acl) use ($resource) : bool {
                return $acl->canAccess($resource);
            }
        );
    }

}
 ```
 in the above example the law of demeter is violated because we allow acces to the access levels collection (composed in the role) from outside the role entities ( A implicitly requires greater knowledge of object B's internal structure.)
 
 this should be modified to:
 
 ```php
 <?php

class User
{
    public function hasAccessTo(Resource $resource) : bool
    {
        return $this->role->allowsAccessTo($resource)
    }

}
 ``` 
 
 in this way access level collection interation is done in the entity where that state exist.
 
 * Entity validity
    * Entities should always be in a valid state after __construct
    * Invalid state should be treated in a different object 
        * Maybe you need a DTO (data transfer object)
    * Use named constructor fot that (e.g. http://verraes.net/2014/06/named-constructors-in-php/)
    * since we keep the state valid and avoid interraction about state (stateless) -> that will lead to avoiding setter methods
    * Avoid coupling with the application layer (framework: zend, laravel etc)
    
  bad example of entity validity:
  
  ```php
<?php

class UserController
{
    public function registerAction()
    {
        $this->userForm->bind(new User());
    }

}
  ```
the first violation of this code is the creation of the User entity -> because no parameters have been passed to the constructor -> this violate the rule: after __construct the entity should be valid.

the second violation: the form interract directly with the state of the entity -> in this cases the programmer start to modify the user entity to make the form work. Instead of adapting the form to work with the user entity, they do the opposite -> they start to create setter and getter in the user to make it work with the form. 
  
second bad example of entity validity:

```php
<?php

class UserController
{
    public function registerAction()
    {
        $this->em->persist(User::fromFormDate($this->form));
    }

}
```
first violation: tightly coupling the core entity (User) with the Form (that come from the framework) -> form components break entity validity

* avoid autogenerated identifiers:
    * if you use -> insert will block eachother
    * if you use -> the entity will not have an id until the entity is persisted
    * if you use -> your object cannot work without the database
    * solution: use UUID (128 bit integer) -> you can store it in mysql and oracle


* query functions are better than repositories

before:
```php
final class UserRepository
{
    public function findUsersThatHaveAMonthlySubscription()
    {
        // SQL HELL HERE ..
    }
}
```

after:

```php
final class UsersThatHaveAMonthlySubscription
{
    public function __construct(
        EntityManagerInterface $em
    )
    {
        // ...
    }

    public function __invoke() : Traversable
    {
        // SQL HELL HERE ...
    }
}
```

* repositories are services, so the query functions -> because of that avoid to use ObjectManager::getRepository(), you should inject it as dependency in your service classes

    
* favour immutable entities (or append-only data structure) 
    * immutable data is simple
    * immutable data is cacheable (forever)
    * immutable data is predictable
    * immutable data enables historical analysis   
    
```php
<?php

class PrivateMessage
{

    private $from;

    private $to;

    private $message;

    private $read = [];

    public function __construct(
        User $from,
        User $to,
        string $message
    )
    {
        // ...
    }

    public function read(User $user)
    {
        $this->read[] = new MessageRead($user, $this);
    }
}



class MessageRead
{
    
    private $user;
    
    private $message;
    
    public function __construct(
        User $user,
        Message $message
    )
    {
        $this->id = UUID::uuid4();
        $this->user = $user;
        $this->message = $message;
    }
}
```    

* Communicate between boundaries (modules or services) via identifiers, not object references
    * keep transaction unrelated: use ObjectManager#clear() between different ObjectManager#flush() calls  
    * that avoid strage state condition -> you don't know if you can flush -> you don't know who (what service) is responsible for flushing 
    
* lifecycle callback
    * don't implement bussiness logic inside lifecycle callback    