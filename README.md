# A brief introduction to Java IDL

This short example will give you a basic introduction to Java IDL, Java's technology for distributed objects based on CORBA. **The example code will be added to this repository later in the week**.

The example is of a shared Address Book running on a server, and we'll implement functionality to:
  1. Query the address book from a remote client;
  2. Add new contact information from a remote client;
  3. Illustrate the Dynamic Skeleton Interface: servicing a request server-side without creating a servant object;

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

3\. Open up a terminal and navigate to the **src-generated** directory. From here, we're going to generate the IDL mapping to Java using the idlj compiler, like so:

```
> idlj -fall addressbook.idl
```

4\. Using the *-fall* flag instructs the compiler to generate both the client-side stub and server-side skeleton mappings along with the AddressBook interface and helper classes.


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

## Implementing the server

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
			
			// 2. Get reference to root POA and activate the POA manager.
			POA rootpoa = POAHelper.narrow(orb.resolve_initial_references("RootPOA"));
			rootpoa.the_POAManager().activate();

			// 3. Create servant and register it with the ORB.
			AddressBookImpl addressBookImpl = new AddressBookImpl();
			org.omg.CORBA.Object ref = rootpoa.servant_to_reference(addressBookImpl);
			AddressBook addressBookRef = AddressBookHelper.narrow(ref);

			// 4. Bind reference with NameService.
			NamingContext namingContext = NamingContextHelper.narrow(orb.resolve_initial_references("NameService"));
			NameComponent[] nc = { new NameComponent("AddressBook", "") };
			namingContext.rebind(nc, addressBookRef);

			// 5. Run the ORB and wait for client calls.
			System.out.println("Address Book Server running...");
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
			AddressBook addressBook = AddressBookHelper.narrow(adressBookRef);
            System.out.println("Obtained a handle on Address Book object...");

			// 4. Execute queries.
			System.out.println(addressBook.lookupEmailFromName("Bob"));
			System.out.println(addressBook.lookupNameFromEmail("mary@example.com"));
		} catch (Exception e) {
			System.out.println("ERROR : " + e);
			e.printStackTrace(System.out);
		}
	}
}
```

## Putting it together

 1\. First, we need to run the Java IDL Object Request Broker Daemon to allow us to register and lookup objects. By default, the daemon will create the directory *orb.db* for its persistent storage. Start the daemon using the following command.

```
> orbd -port 1049 -ORBInitialHost localhost -ORBInitialPort 1050 
```

2\. From another terminal / command prompt, navigate to the project **bin** directory, and start the server.

```
> java org.example.corba.Server
```

3\. Finally, from another terminal window, navigate again to the project **bin** directory and start the client.

```
> java org.example.corba.Client
```

If all goes well, you should see the results of the queries printed to the console.

## Now it's your turn...

In the remainder of the class we'll develop and discuss how to extend the example by carrying out the following tasks.

 1. Add an *addContact(...)* method to the address book interface, to support the addition of new contacts through the passing a name and email address as parameters. Modify the client to use this interface and check it works by using the query methods as before.

 2. Modify the IDL file to create a **Person** interface, with accessors for name and email address. Use the same approach as for the AddressBook, to create a simple implementation (**PersonImpl.java**).

 3. Modify the *addContact(...)* method from step 1 to accept a **Person** object as a parameter instead. Make the necessary updates to the **Client** to use this method.  Within the **Client** you will need to use the *rootPOA.servant_to_reference(...)* method, followed by *PersonHelper.narrow(...)* on the result to obtain an object of type **Person**.

(Extra: You may wish to investigate how to realise an *addPeople(...)* method, using the IDL concept of a *sequence*).

Your final **Client** class should have a structure similar to the following.

```java
package org.example.corba;

public class Server {

	public static void main(String args[]) {
		try {
			// 1. Set server location and initialise ORB.
			
			// 2. Get the root naming context
			
			// 3. Lookup reference to the Address Book.
			
			// 4. Execute queries.
			
			// 5. Get reference to root POA and activate the POA manager.
			
			// 6. Create new people	and resolve to `Person' references		
			
			// 7. Add people using the addPerson() and addPeople() methods.
			
			// 8. Query for details of the newly added people.
		} catch (Exception e) {
			// Handle Exceptions.
		}
	}
}
```

## Running the example using multiple ORBs (on multiple machines)

Throughout this exercise we've developed all the examples using a reference to a single, locally running ORB. If you've time (or in your spare time), team up with somebody to run the examples across multiple machines.

Ensure that the **AddressBook** and **Person** objects are registered on different ORBs (running on different machines). You'll need to modify the **Client** code to achieve this.

For development purposes, you can start two ORBs on a single machine as follows:

```
> orbd -port 1049 -ORBInitialHost localhost -ORBInitialPort 1050 &
> orbd -port 1051 -ORBInitialHost localhost -ORBInitialPort 1052 &
```

## Using IORs directly

Instead of using the Name Service, you may want to resolve a reference to a distributed object directly from its IOR.

1. Modify the **Server** class to comment out the lines that register the Address Book with the Name Service. Instead, print out the Address Book's IOR to the console. You'll need to use the following method.

```java
orb.object_to_string(addressBook);
```

2. Modify the **Client** class to accept an IOR as an argument from the command line. Comment out the lines that lookup the Address Book via the Naming Service (or add a control statement), and instead use the IOR to resolve the reference.

```java
serverOrb.string_to_object(ior);
```


## A final exercise... using DSI

Finally, we'll illustrate an alternative to implementing the Servant class without that uses the Dynamic Skeleton Interface (DSI) instead of the POA model.  DSI allows us to provide an implementation of the Address Book without any of the types generated from the mapping (**Person**, **UnknownNameException**, etc.). This would be useful had we to implement the server in a language with no concept of objects.

The example given below provides a basic implementation of *lookupEmailFromName()* method. It's useful to work through it to make sure you understand what's going on. If you want to take the next steps to implement the other methods, you'll likely need to spend some time familiarising yourself with the API.

Carry out the following steps:

1. Create a new class **DSIAddressBookImpl**, using the template below.

```java
package org.example.corba;

public class DSIAddressBookImpl extends DynamicImplementation {

	/**
	 * Map from names to email addresses.
	 */
	private Map<String, String> contacts = new HashMap<String, String>();

	/**
	 * Reference to the ORB that manages this object.
	 */
	private ORB orb;

	/**
	 * Constructor: Adds some default entries to the address book.
	 */
	public DSIAddressBookImpl(ORB orb) {
		this.orb = orb;
		contacts.put("Bob", "bob@example.com");
		contacts.put("Alice", "alice@example.com");
		contacts.put("Mary", "mary@example.com");
	}

	/**
	 * Receives all requests and handles dispatch to handler methods.
	 * 
	 * @param req
	 *            the server request.
	 */
	public void invoke(ServerRequest req) {
		// Extract method name from request
		String op = req.operation();
		
		// Use a case statment to dispatch the request to a handler method.
		switch (op) {
		case "lookupEmailFromName":
			methodLookupEmailFromName(req);
			break;
		// case "lookupNameFromEmail":
		//    methodLookupnameFromEmail(req);
		//    break;
		default:
			throw new org.omg.CORBA.BAD_OPERATION();
		}
	}

	/**
	 * Processes a server request to lookup an email address from a name.
	 * 
	 * @param request
	 *            the server request.
	 */
	private void methodLookupEmailFromName(ServerRequest request) {
		
		// We expect a single argument of type String
		NVList args = orb.create_list(1);
		Any arg = orb.create_any();
		arg.type(orb.get_primitive_tc(TCKind.tk_string));
		args.add_value("", arg, ARG_IN.value);
		
		// Extract the argument from the request
		request.arguments(args);
		String name = arg.extract_string();
		
		// Create the result object
		Any result = orb.create_any();
		
		// Execute the operation
		if (contacts.containsKey(name)) {
			String email = contacts.get(name);
			// Set the result 
			result.insert_string(email);
			request.set_result(result);
		} else {
			// Raise an exception (without use of the generated exception class)
			StructMember[] members = new org.omg.CORBA.StructMember[0];
			TypeCode typeCode = org.omg.CORBA.ORB.init().create_exception_tc(
					"IDL:org/example/corba/UnknownNameException:1.0",
					"UnknownNameException", members);
			OutputStream out = result.create_output_stream();
			out.write_string("IDL:org/example/corba/UnknownNameException:1.0");
			result.type(typeCode);
			result.read_value(out.create_input_stream(), typeCode);
			request.set_exception(result);
		}
	}
	
	/**
	* The intefaces implemented by this DSI implementation.
	*/
	public String[] _all_interfaces(POA poa, byte[] oid) {
		return new String[] { "IDL:org/example/corba/AddressBook:1.0" };
	}
}
```

2\. Modify the **Server** class to create an instance of the **DSIAddressBookImpl** class, and register it with the Name Service using the name "DSIAddressBook"

3\. Modify the **Client** class to lookup "DSIAddressBook" instead of the standard address book.

4\. In the Client class, comment out all calls to the AddressBook object, with the exception of the initial call to *lookupEmailFromName()*.

5\. Run the example and check that the correct result is returned.

6\. If you feel inclined to do so, modify **DSIAddressBookImpl** to implement the remaining methods. Uncomment the relevant lines in the **Client** as you go to test that your implementation works.  In carrying this out you'll likely need to research Java's DSI API in more detail. 




