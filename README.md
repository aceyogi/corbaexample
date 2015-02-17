# A brief introduction to Java IDL

This short example will give you a basic introduction to Java IDL, Java's technology for distributed objects based on CORBA.

The example is of a shared Address Book running on a server, and we'll implement functionality to:
  1. Add new contact information from a remote client;
  2. Query the address book from a remote client;
  3. Illustrate the Dynamic Skeleton Interface: servicing a request server-side without creating an object;
  4. Listen for updates to the address book using the Observer pattern.

The setup described below assumes that you're using Eclipse. 

## Setup
1. Create a new Java project.
2. Create a directory: **src-generated** in addition to your **src** directory.  We'll use this to separate the generated code from the code we write ourselves.
3. Add the new folder to the build path (right click: 'build path' > 'use as source folder')
4. Setup the package *org.example.corba* in the **src** directory, and mirror this in the **src-generated** directory.

Your project directory should look something like:

![Directory Structure](http://i.imgur.com/RGYwmH4.png)

## A basic IDL file
We use IDL to define interfaces for distributed objects, which is used by the **idlj** compiler to translate the definitions into Java.  

1. Create a file **addressbook.idl** in the **src-generated** directory. We'll use this file to define all the object interfaces in our example.

2. First, lets define a simple interface for an Address Book that allows contacts to be queried by name or email address (with exceptions thrown for incorrect input). Notice that the Java package structure we defined corresponds to nested module definitions.

```
module org {
   module example {
      module corba {
      
         exception UnknownNameException {};
         exception UnknownEmailAddressException {};
      
         interface AddressBook {
            string lookupEmailFromName(in string name) raises (UnknownNameException);
            string lookupNameFromEmail(in string email) raises (UnknownEmailAddressException);
         };

      };        
   };          
};
```

3. Open up a terminal and navigate to the **src-generated** directory. From here, we're going to generate the IDL mapping to Java using the idlj compiler, like so:

```
idlj addressbook.idl

```

4. Using the *-fall* flag instructs the compiler to generate both the client-side stub and server-side skeleton mappings along with the AddressBook interface and helper classes.


## Implementing the Address Book

In the **src** directory, create a new class, **AddressBook.java**, in the *org.example.corba* package. We provide an implementation for the Address Book (a servant) by extending from the generated **AddressBookPOA** skeleton class (which, in turn, implements the **AddressBook** interface).

```java
package org.example.corba;

/**
 * An implementation of the Address Book.
 */
public class AddressBookImpl extends AddressBookPOA {

	/**
	 * Map from names to email addresses.
	 */
	private Map<String, String> contacts = new HashMap<String, String>();

	/**
	 * Constructor: Adds some default entries to the address book.
	 */
	public AddressBookImpl() {
		contacts.put("Bob", "bob@example.com");
		contacts.put("Alice", "alice@example.com");
		contacts.put("Mary", "mary@example.com");
	}

	/**
	 * Check the contacts map for an entry with the given name. If it exists,
	 * return the associated email address, otherwise raise an exception.
	 */
	public String lookupEmailFromName(String name) throws UnknownNameException {
		if (contacts.containsKey(name)) {
			return contacts.get(name);
		} else {
			// Name not known, so raise exception.
			throw new UnknownNameException();
		}
	}

	/**
	 * Check the contacts map for an entry with the given email address. If it
	 * exists, return the associated name, otherwise raise an exception.
	 */
	public String lookupNameFromEmail(String email)
			throws UnknownEmailAddressException {
		for (Entry<String, String> entry : contacts.entrySet()) {
			if (email.equals(entry.getValue())) {
				return entry.getKey();
			}
		}
		// No matching entry found, so raise exception.
		throw new UnknownEmailAddressException();
	}
}
```

## Implementing the Server

Next, we need to write a server class, which is responsible for:
   1. Creating the Object Request Broker (ORB) and activating the POA Manager.
   2. Creating instances of the servant classes (here, the **AddressBookImpl**), and registering them with the ORB.
   3. Optionally, registering the servant classes under a given name, using the root naming context (alternatively, the Interoperable Object Reference (IOR) can be used directly).

Create **Server.java** as follows:

```java
package org.example.corba;

public class Server {

	public static void main(String args[]) {
		try {
			// 1. Set server location and initialise ORB.
			System.out.println("Address Book Server starting...");
			Properties props = System.getProperties();
			props.put("org.omg.CORBA.ORBInitialPort", "1050");
			props.put("org.omg.CORBA.ORBInitialHost", "localhost");
			ORB orb = ORB.init(args, props);
			System.out.println("Initialized ORB");

			// 2. Get reference to root POA.
			POA rootpoa = POAHelper.narrow(orb.resolve_initial_references("RootPOA"));

			// 3. Create servant and register it with the ORB.
			AddressBookImpl contactsImpl = new AddressBookImpl();
			org.omg.CORBA.Object ref = rootpoa.servant_to_reference(contactsImpl);
			AddressBook contactsRef = AddressBookHelper.narrow(ref);

			// 4. Bind reference with NameService.
			NamingContext namingContext = NamingContextHelper.narrow(orb.resolve_initial_references("NameService"));
			NameComponent[] nc = { new NameComponent("AddressBook", "") };
			namingContext.rebind(nc, contactsRef);

			// 5. Run the ORB and wait for client calls.
			System.out.println("Address Book Server running...");
			rootpoa.the_POAManager().activate();
			orb.run();
		}

		catch (Exception e) {
			System.err.println("ERROR: " + e);
			e.printStackTrace(System.out);
		}

		System.out.println("Address Book Server Exiting ...");
	}
}
```

## Implementing the client

Next, we need to write a client class, which will:

 1. Use the Name Service of the ORB to obtain a reference to the AddressBook object.
 2. Query the Address Book.

```java
package org.example.corba;

public class Client {

	public static void main(String args[]) {
		try {
			// 1. Set server location and initialise ORB.
			Properties props = System.getProperties();
			props.put("org.omg.CORBA.ORBInitialPort", "1050");
			props.put("org.omg.CORBA.ORBInitialHost", "localhost");
			ORB orb = ORB.init(args, props);
			System.out.println("Initialized ORB");

			// 2. Get the root naming context
			org.omg.CORBA.Object objRef = orb.resolve_initial_references("NameService");
			NamingContextExt ncRef = NamingContextExtHelper.narrow(objRef);
			
			// 3. Lookup reference to the Address Book.
			org.omg.CORBA.Object adressBookRef = ncRef.resolve_str("AddressBook");
			AddressBook contactsImpl = AddressBookHelper.narrow(adressBookRef);
            System.out.println("Obtained a handle on Address Book object...");

			// 4. Execute queries.
			System.out.println(contactsImpl.lookupEmailFromName("Bob"));
			System.out.println(contactsImpl.lookupNameFromEmail("mary@example.com"));
		} catch (Exception e) {
			System.out.println("ERROR : " + e);
			e.printStackTrace(System.out);
		}
	}
}
```

## Putting it Together

 1. First, we need to run the Java IDL Object Request Broker Daemon to allow us to register and lookup objects. Start the daemon using the following command.

```
orbd -ORBInitialPort 1050 -ORBInitialHost localhost
```

2. From another terminal / command prompt, navigate to the project **bin** directory, and start the server.

```
java org.example.corba.Server
```

3. Finally, from another terminal window, navigate again to the project **bin** director and start the client.

```
java org.example.corba.Client
```

If all goes well, you should see the results of the queries printed to the console.

## Now it's your turn...

In the remainder of the class we'll develop and discuss how to extend the example by carrying out the following tasks.

 1. Add an *addContact(...)* method to the address book interface, to support the addition of new contacts through the passing a name and email address as parameters. Modify the client to use this interface and check it works by using the query methods as before.

 2. Modify the IDL file to create a **Person** interface, with accessors for name and email address. Using the same approach as for the AddressBook, create a simple implementation (**PersonImpl.java**).

 3. Modify the *addContact(...)* method from step 1 to accept a **Person** object as a parameter instead. Make the necessary updates to the client to use this method.  Within the client you will need to use the *rootPOA.servant_to_reference(...)*, followed by *PersonHelper.narrow(...)* to obtain an object of type **Person**.


## As example using DSI.