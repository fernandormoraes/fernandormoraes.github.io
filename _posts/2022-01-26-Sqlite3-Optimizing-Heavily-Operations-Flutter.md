# Working with SQLite Heavily operations in Flutter(Trasanction, PreparedStatement and Isolates)

Hi everyone, last week i was working with a problem in an flutter application, we need our application inserting a lot of data in sqlite from a JSON and i manage to solve this issue using some basic operations, but a important thing is that i used Isolates too, so in this article i'll show how i face the problem and develop a workaround with sqlite3 ffi lib provided by Simon Binder package as the package wasnt designed with Isolates -as sqflite and drift was-.


## SQLite3 optimized inserts

Following this article about improve sqlite inserts https://stackoverflow.com/questions/1711631/improve-insert-per-second-performance-of-sqlite we noticed that inserts in a transaction and using prepared statement's will obviously improve performance.

*Why??* When inserting rows in a sqlite database outside a transaction every insert has to make his own transaction and this implies locking and unlocking data to prevent consistency problems, so inserting this data inside a unique transaction will dramatically improve performance depending on the amount of data, plus the prepared statement will prevent sqlite from validating the insert query every time it happens, i wrote a example below to illustrate what exactly i did:

```
    var  db  =  sqlite3.open(dbPath);

	final stmt = db.prepare('INSERT INTO PRODUCT (description, price, category) VALUES(?, ?, ?); ');
	
	db.execute('BEGIN TRANSACTION; ');

	for (Product product in listProducts) {
		stmt.execute([product.description,product.price,product.category]);
	}
	
	stmt.dispose();

	db.execute('COMMIT;');
	db.dispose
```

## Isolating the heavy work

*Why should i use isolate?* I strongly recommend [this](https://dev.to/alphamikle/why-should-you-use-isolates-in-flutter-1k5o) article for understand Isolate and why use it.

Using transaction and prepared statement surely will make the inserts fast as possible, but in this way our flutter application will get stuck frames due to processing everything, so we gonna make what sqflite and drift does using another thread with Isolate class.

I was really inspired by how sqflite used isolate, so the code similarity is not a coincidence since i copied their solution lmao, Isolate in dart is not the best intuitive implementation and i really liked how descriptive is their code documentation and how the method works.

```
import  'dart:isolate';
import  'package:example/Models/Products.dart';
import  'FfiMethodCall.dart';

/// Signature responsible for overriding the SQLite dynamic library to use.
/// Adapted to work with my method passing a List<Product> parameter.
typedef  SqliteFfiInit  =  void  Function(List<Product> params);

/// Sqlite isolate.
class  SqliteIsolate {
	SqliteIsolate({this.sendPort});

	/// Our send port.
	final  SendPort  sendPort;
}

Future<SqliteIsolate> createIsolate(
	SqliteFfiInit ffiInit, List<dynamic> ffiParams) async {

	// create a long-lived port for receiving messages
	var  ourFirstReceivePort  =  ReceivePort();

	// spawn the isolate with an initial sendPort.
	await  Isolate.spawn(
		_isolate, [ourFirstReceivePort.sendPort, ffiInit, ffiParams]);
		
	// the isolate sends us its SendPort as its first message.
	// this lets us communicate with it. we’ll always use this port to
	// send it messages.
	var  sendPort  = (await  ourFirstReceivePort.first) as  SendPort;

	return  SqliteIsolate(sendPort:  sendPort);
}

/// The isolate
Future  _isolate(List<dynamic> args) async {

	// open our receive port. this is like turning on
	// our cellphone.
	var  ourReceivePort  =  ReceivePort();

	final  sendPort  =  args[0] as  SendPort;
	final  ffiInit  = (args[1] as  SqliteFfiInit);
	final  ffiParams  =  args[2];

	// Call function passed in SqliteFfiInit instance with createIsolate params
	ffiInit?.call(ffiParams);

	// tell whoever created us what port they can reach us on
	// (like giving them our phone number)
	sendPort.send(ourReceivePort.sendPort);

}
```

Usage is pretty simple in this case since we "hardcoded" the object we pass in the list since my use case is very specific it was not necessary to do generics class/methods.

``` 
Future<void> insertProducts(List<Product> listProducts) async {
	// insertAll is our method which will be isolated
	await createIsolate(insertAll, listProducts);
}

void  insertAll(List<Product> listProducts) async {
	// Here is where we use our code who made all the inserts as the example above
}
```

## Conclusion

After that our flutter application will process the inserts faster enough and we still gonna have a good performance in UI side, any suggestion and better solutions i would appreciate.

