# test_my_repo
I need to write a script for an SSIs package, targeted at sql server2019. 

The goal is to identify a file from a folder whose name matches three conditions :

1. The beginning of the. File name must match a string provided by variable. 
2. The provided string must be immediately followed by an underscore and a string of exactly 14 numerals. 
3. The file must have a file extension that matches a file extension also provided in a variable. It may also optionally include a second extension in the last position of the file file name. Acceptable file names will end with the first extension, or with the first  extension followed immediately by a period and then the second extension.  

When all matching files have been gathered into an array or other structure, I need it to return the full name of the file into a dts variable. I also want a second dts variable containing the file name with its first extension only. 