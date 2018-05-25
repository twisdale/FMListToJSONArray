FMListToJSONArray ( key ; fmList )

Usage: 	 FileMaker 16 Pro Advanced and later, FileMaker List, JSON Array, Javascript

Purpose: Translate a FileMaker 16 list into a proper JSON Array without iterative 
	looping, a capability that custom functions do not have, or recursion.
	In other words, this function creates a JSON array, either named or unnamed,
	given a key and a FileMaker list.			 

Discussion:
	There is a difference between a FileMaker list and a JSON array, which may or may not be apart of a JSON object. However, there is no native function to translate a FileMaker list into a JSON array. 

	---------
	FileMaker
	---------

	In FileMaker, the values of a List are delimited by a Carriage return, '\r', 0x0D, which we will refer to as ("CR"). A list can be defined in three ways

		1. List ("Ford"; "BMW"; “Fiat”)
		2. "Ford¶BMW¶Fiat"  (i.e. "Ford\rBMW\rFiat")

	these two expressions are identical to FileMaker.
				 
	The third way to define a list is using the valuelist capabilities of the system. The other lists are essentially variations of these three. 
	(For the purpose of this discussion, we are deliberately not going into valueless used for pop up or drop down menus, etc.)
				 
	Lists can be assigned to a variable 
		
		Set Variable ($cars; "Ford¶BMW¶Fiat") 
	
	or a field
		
		Set Field (TableName::SomeField; "Ford¶BMW¶Fiat")

	The elements of the list can be inspected by using FileMaker functions RightValues or LeftValues both of which return a CR at the end of the result.
	for example, 
		
		RightValues ( "Ford¶BMW¶Fiat" ; 1 )  & "Test"  returns:
			Fiat
			Test
				 
	NOTE: "Fiat" does not contain a CR in the definition. That is we see "Fiat" without a CR not "Fiat¶" with a CR. Yet RightValues adds CR when returning the value requested for any index.
				 
	Alternatively, we can access the list using the function GetValue which strips out the CR regardless of whether there was an CR in the definition for that index or not.
		
		GetValue ( "Ford¶BMW¶Fiat" ; 1 ) & "-Test #2" returns
			Ford-Test #2


	------------------------
	JSON (see W3schools.com)
	------------------------
			 
	In JSON, arrays are almost always the same as arrays in Javascript. They are values delimited by a comma not a CR, as in FileMaker. 
			 
	An unnamed array of the above would be given by,
		[ "Ford", "BMW", "Fiat" ]
				
	Also, an array can be a value of an object property.
		{
			“owner”:"
			"cars":[ "Ford", "BMW", "Fiat" ]
		}
			 
	Just as FileMaker array values would be accessed by index using the GetValue function, so too, JSON array values are accessed by an index using brackets, for example,
		
		cars[0] & "-Test #2" 
	
	returns
		
		Ford-Test #2
					
	Index values in JSON start at 0, while in FileMaker they start at 1.

	----------------------------------
	Translation from FileMaker to JSON 
	----------------------------------
				
	Why is all of this important? FileMaker 16 has new native capabilities to interface with JSON / Javascript. However, there is no native function to translate a FileMaker list into a JSON array.
			
	Also, FileMaker JSON function, JSONSetElement, could easily be mistaken to produce a JSON array from a FileMaker list. We would then expect JSONListKeys and JSONListValues to produce results that could be operated on as JSON objects. But that is not the case. 


	For example, suppose we set the field JSONTest::car to "Ford¶BMW¶Fiat" as follows:

		Set Field[JSONTest::car; "Ford¶BMW¶Fiat"] 
			
	One might attempt to translate "Ford¶BMW¶Fiat" into a JSON array by the following:

		JSONSetElement ( “{}"; ["car"; JSONTest::car ; JSONString] )
					
	But this would produce a Property, with its value being a string, not an array

		{“car”:”Ford\rBMW\rFiat"}

	If we apply JSONListValues, with NULL keyOrIndexOrPath as follows, 

		JSONListValues (
			JSONSetElement ( “{}”; ["car"; JSONTest::car ; JSONString] ) 
		; “” ) 

	then it returns the value, i.e. "Ford\rBMW\rFiat", in FileMaker format with the "\r" translated giving us the List:

		Ford
		BMW
		Fiat

	But this value is not an indexable value from JSON's perspective even though we can apply the GetValue function from FileMaker. for example.

		GetValue (
			JSONListValues ( 
				JSONSetElement ( “{}”; ["car"; JSONTest::car ; JSONString] ) 
			; “” )
		; 2 ) 

	returns

		BMW

	But we could not use JSONListValues to get the nth value. I.e.

		JSONListValues ( 
			JSONSetElement ( “{}"; ["car"; JSONTest::car ; JSONString] )
		; i ) 

	is null for 0, 1 , 2 and also for "car".


	The correct way to create a proper JSON array from this list would be as follows:
			
		JSONSetElement ( "{}";
			[“car[0]”; "Ford"; JSONString] ;
			[“car[1]”; "BMW"; JSONString] ;
			[“car[2]”; "Fiat"; JSONString] 
		)
					
	This will produce exactly what we are looking for:
		{
			"car" : [ "Ford", “BMW", “Fiat"]
		}

	Now, JSONListKeys on this JSON array will produce three keys: 
		0
		1
		2

	And JSONListValues will produce three values: 
		Ford
		BMW
		Fiat

	More generally, we want to take a FileMaker list with an arbitrary number of list values, where the key or the values may be Null.

		JSONSetElement ( “{}";
			[“car[0]”; GetValue(JSONTest::car; 1); JSONString] ;
			[“car[1]”; GetValue(JSONTest::car; 2); JSONString] ;
			… 
			[“car[N-1]”; GetValue(JSONTest::car; N); JSONString]
		)

	This produces a proper JSON Array:
		{
			"car" : [ "car_1", “car_2", ...., “car_N"]
		}

	JSONListKeys will produce a list of N keys
		0
		1
		…
		N-1
					
	And JSONListValues will produce a list of N values:

		car_1
		car_2
		…
		car_N
				
				
	Doing this manually would be not only tedious but impossible if the information is arbitrary.
				
	FMListToJSONArray is a function to automate the translation from a FileMaker list to a JSON array. 

	It does not iterate or recurse. 

	It handles both a null key ="" and null list values, such as 

		"", or "¶", or "Ford", or "Ford¶" or "¶Ford" .
				
	Examples:

		1.	FMListToJSONArray ( "car" ; "" ) 		output 	{“car”:[""]} 
			— JSON object with one property, an Array
		2.	FMListToJSONArray ( "" ; "" ) 			output 	[“”] 
			— JSON empty array
		3.	FMListToJSONArray ( "car" ; "¶" ) 		output 	{“car”:["",""]}
			— JSON object with one property, a sparse array
		4.	FMListToJSONArray ( "" ; "¶" ) 			output 	[“",""]
		5.	FMListToJSONArray ( "car" ; "Ford" ) 	output 	{“car":["",""]}
		6.	FMListToJSONArray ( "" ; "Ford" ) 		output 	{“car":["Ford"]}
		7.	FMListToJSONArray ( "car" ; "Ford¶" ) 	output 	{“car":["Ford",""]}
		8.	FMListToJSONArray ( "" ; "Ford¶" ) 		output 	[“Ford",""]
		9.	FMListToJSONArray ( "car" ; "¶Ford" ) 	output 	{“car":["","Ford"]}
		10.	FMListToJSONArray ( "" ; "¶Ford" ) 		output 	[“","Ford"]

	The rest is obvious. 

	NOTE: the CR indicates that there is a value before the CR and one after the CR, even though it might be Null.

	-----------------------------------------	
	Differences between FMList and JSON Array
	-----------------------------------------	

	Using this function you can create a sparse array. For Example:

		FMListToJSONArray ( "car" ; "¶Ford¶¶BMW¶" )

	outputs:

		{"car":["","Ford","","BMW",""]}

	We can then update the values in the array to the following: 

		{"car":["Chevy BOLT","Ford","BMW",""]}

	As follows:

		JSONSetElement ( 
			FMListToJSONArray ( "car" ; "¶Ford¶¶BMW¶" ) ; 
		"car[0]" ; "Chevy BOLT" ; JSONString ) 

	outputs:

		{"car":["Chevy BOLT","Ford","","BMW",""]}
				
	If we create a FileMaker list using the result from JSONListValues on the sparse array we get a ‘compacted' list with three values instead of 5, meaning that the CRs were removed:

		JSONListValues ( 
			FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) 
		; "car" )

	outputs:
		Chevy BOLT
		Ford
		BMW

	We can confirm this by looking at the ValueCount of the list

		ValueCount ( 
			JSONListValues ( 
				FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) 
			; "car" ) 
		) 

	outputs:
		3
				
	But JSONListKeys returns the entire set of keys for the array:

		JSONListKeys ( 
			FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) 
		; "car" )

	outputs:
		0
		1
		2
		3
		4
				
	with a ValueCount of 5. This allows us to recreate the sparse array as a FileMaker list if necessary. 

	The object created using the JSON function set can be passed to a Javascript system seamlessly. It seems that this is what FileMaker developers had in mind.

