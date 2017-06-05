# FMListToJSONArray
Purpose: Translate a Filemaker list into a proper JSON Array without iterative looping, a capability that custom functions do not have, or recursion.
		 In other words, this function creates a JSON array, either named or unnamed, given a key and a Filemaker list.


Discussion:
			 There is a difference between a Filmmaker list and a JSON array, which may or may not be apart of a JSON object. 

			 ---------
			 FILEMAKER
			 ---------

			 In Filemaker, the values of a List are delimited by a Carriage return, '\r', 0x0D, which we will refer to a ("CR"). A list can be defined in three ways
			 1) List ("Ford"; "BMW"; "Fist")
			 2) "Ford¶BMW¶Fiat"  (i.e. "Ford\rBMW\rFiat")
			 these two expressions are identical to Filemaker.
			 
			 The third way to define a list is using the value list capabilities of the system. The other lists are essentially variations of these three. 
			 (For the purpose of this discussion, we are diliberately not going into pop or drop down menus, etc.)
			 
			 Lists can be assigned to a variable 
			 	Set Variable ("cars"; "Ford¶BMW¶Fiat") 
			 or a field
			 	Set Field (TO:SomeField; "Ford¶BMW¶Fiat")
			 
			 The elements of the list can be inspected using by using Filemaker functions RightValues or LeftValues both of which which return a CR at the end of the result.
			 for example, 
				 RightValues ( "Ford¶BMW¶Fiat" ; 1 )  & "Test"  returns:
				 Fiat
				 Test
			 
			 NOTE: Fist does not contain a CR in the definition, yet RightValues adds it when returning the value requested for any index.
				 
			Alternatively, we can access using the function GetValue which strips out the CR regardless of whether there was an CR in the definition for that index or not.
			 GetValue ( "Ford¶BMW¶Fiat" ; 1 ) & "-Test #2" returns
			 	Ford-Test #2

			 ------------------------ 
			 JSON (see W3schools.com)
			 ------------------------
			 
			 In JSON, arrays are almost always the same as arrays in Javascript. They are values delimited by a comma not a CR. 
			 
			 An unnamed array of the above would be given by,
			 	[ "Ford", "BMW", "Fiat" ]
				
			Also, an array can be a value of an object property.
				{
				"owner":"John",
				"cars":[ "Ford", "BMW", "Fiat" ]
				}
			 
			 Just as Filemaker array values would be accessed by index using the function GetValue, so to are JSON array values accessed by an index using brackets, [index].
				cars[0] & "-Test #2" returns
			 		Ford-Test #2
					
			index values in JSON start at 0, while in FIlemaker they start at 1.
			
			----------------------------------
			Translation from Filemaker to JSON 
			----------------------------------
			
			Why is all of this important? Filemaker 16 has new native capabilities to interface with JSON / Javascript. However, there is no native function to translate a Filemake rlist into a JSON array.
			
			Also, Filemaker's new JSON function JSONSetElement could easily be mistaken to produce  JSON array from a Filemaker list. We would then expect
			JSONListKeys and JSONListValues to produce results as if operating on JSON objects. But that is not the case. 
			
			For example, one might attempt to translate "Ford\rBMW\rFiat" into a JSON array by the following:
			
				JSONSetElement ( "{}";
				 ["car"; JSONTest::car ; JSONString]
				)
				
				But this would produce not an array but a Property, with a value of a string 
					{“car”:"Ford\rBMW\rFiat"}
			
			Using ListValues would then produce a line count not a count of indexable values.
			
			The correct way to create a proper JSON array from this list would be as follows:
			
				JSONSetElement ( "{}";
				[“car[0]”; "Ford"; JSONString] ;
				[“car[1]”; "BMW"; JSONString] ;
				[“car[2]”; "Fiat"; JSONString] 
				)
				
			Which would produce exactly what you are looking for:
					{
						"car" : [ "Ford", “BMW", "Fiat"]
					}
					
			
			Now ListKeys on this JSON object will produce three Keys:
				0
				1
				2
			
			And ListValues will produce three Keys:
				Ford
				BMW
				Fiat
			
			More generally, the function we want will take a Filemaker list with an arbitrary number of list vales, where the key or the values may be Null.
			
				JSONSetElement ( "{}";
				[“car[0]”; GetValue(JSONTest::car; 1); JSONString] ;
				[“car[1]”; GetValue(JSONTest::car; 2); JSONString] ;

				… 
				[“car[N-1]”; GetValue(JSONTest::car; N); JSONString] 
				)
			
			It will then produce the correct proper JSON Array:
				{
					"car" : [ "car_1", “car_2", ...., "car_N"]
				}

			ListKeys will produce a list of N keys
				0
				1
				...
				N-1
				
			And ListValues will produce a list of N Keys:
				car_1
				car_2
				...
				car_N
			
			
			DOing this manually would be not only tedious but impossible if the information is arbirary.
			
			FMListToJSONArray is a function to automate the translation from a Filemaker list to a JSON array. 
			
			It does not iterate or recurse. 
			
			It handles a null key ="" and null list values, such as 
			"", or "¶", or "Ford", or "Ford¶" or "¶Ford" .
			
			Examples:
			1.	FMListToJSONArray ( "car" ; "" ) output
				{"car":[""]}
			2.	FMListToJSONArray ( "" ; "" ) output
				[""]
			3.	FMListToJSONArray ( "car" ; "¶" ) output
				{"car":["",""]}
			4.	FMListToJSONArray ( "" ; "¶" ) output
				["",""]
			5.	FMListToJSONArray ( "car" ; "Ford" ) output
				{"car":["",""]}
			6.	FMListToJSONArray ( "" ; "Ford" ) output
				{"car":["Ford"]}
			7.	FMListToJSONArray ( "car" ; "Ford¶" ) output
				{"car":["Ford",""]}
			8.	FMListToJSONArray ( "" ; "Ford¶" ) output
				["Ford",""]
			9.	FMListToJSONArray ( "car" ; "¶Ford" ) output
				{"car":["","Ford"]}
			10.	FMListToJSONArray ( "" ; "¶Ford" ) output
				["","Ford"]
			
			The rest is obvious. 
			
			NOTE: the CR indicates that there is a value before the CR and one after the CR, even though it might be Null.
			
		-----------------------------------------
		Differences between FMList and JSON Array
		-----------------------------------------	
			Using this function you can create a sparce array. For Example:
				FMListToJSONArray ( "car" ; "¶Ford¶¶BMW¶" )
				outputs:
					{"car":["","Ford","","BMW",""]}
					
			We can then update the values in the array:
			 
			{"car":["Chevy BOLT","Ford","BMW",""]}
				JSONSetElement ( FMListToJSONArray ( "car" ; "¶Ford¶¶BMW¶" ) ; "car[0]" ; "Chevy BOLT" ; JSONString ) 
				outputs:
					{"car":["Chevy BOLT","Ford","","BMW",""]}
			
			If we create a Filemaker list from the sparce array we get a list with three values in 'compacted' order different order, meaning that the spaces were removed:
				JSONListValues ( FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) ; "car" )
				outputs:
					Chevy BOLT
					Ford
					BMW
			We can confirm this by looking at the ValueCount of the list
				ValueCount ( JSONListValues ( FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) ; "car" ) ) 
				outputs:
					3
			
			But JSONListKeys returns the entire set of keys for the array:
				JSONListKeys ( FMListToJSONArray ( "car" ; "Chevy BOLT¶Ford¶¶BMW¶" ) ; "car" ) 
				outputs:
				0
				1
				2
				3
				4
			
			with a ValueCount of 5. 
			
			The object created using the JSON function set can be passed to a Javascript system seamlessly. It seems that this is what Filemaker developers had in mind.
