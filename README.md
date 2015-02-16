# A brief introduction to Java IDL.

This short example will give you a basic introduction to Java IDL, Java's technology for distributed objects based on CORBA.

The example is of a shared Address Book running on a server, and we'll implement functionality to:
  1. Add new contact information from a remote client;
  2. Query the address book from a remote client;
  3. Illustrate the Dynamic Skeleton Interface: servicing a request server-side without creating an object;
  4. Listen for updates to the address book using the Observer pattern.

The setup described below assumes that you're using Eclipse. You can download an eclipse project containing the basic setup from this repository, although it should only take 10 minutes to walk through it.

# Setup (assumes eclipse)
1. Create a new Java project.
2. Create a directory: **src-generated** in addition to your **src** directory.  We'll use this to separate the generated code from the code we write ourselves.
3. Add the new folder to the build path (right click: 'build path' > 'use as source folder')
4. Setup the package *org.example.corba* in the **src** directory, and mirror this in the **src-generated** directory.

Your project directory should look something like:

[image]

# A basic IDL file
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

[img]

4. Using the *-fall* flag instructs the compiler to generate both the client-side stub and server-side skeleton mappings along with the AddressBook interface and helper classes.

5. Next, in your **src** directory, we'll provide an implementation for the Address Book by extending from the generated **AddressBookPOA** class.

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