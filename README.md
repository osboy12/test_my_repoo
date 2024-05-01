# test_my_repo
I need to write a script for an SSIs package, targeted at sql server2019. 

The goal is to identify a file from a folder whose name matches three conditions :

1. The beginning of the. File name must match a string provided by variable. 
2. The provided string must be immediately followed by an underscore and a string of exactly 14 numerals. 
3. The file must have a file extension that matches a file extension also provided in a variable. It may also optionally include a second extension in the last position of the file file name. Acceptable file names will end with the first extension, or with the first  extension followed immediately by a period and then the second extension.  

When all matching files have been gathered into an array or other structure, I need it to return the full name of the file into a dts variable. I also want a second dts variable containing the file name with its first extension only. 

To achieve your goal in an SSIS package for SQL Server 2019, you can use a Script Task component to write a script in C#. This script will access the file system, check for files that match your conditions, and store the results in DTS variables. Here’s how you can structure your script:

	1.	Setup the Script Task: Add a Script Task to your SSIS package.
	2.	Define Variables: Create the necessary variables in SSIS and pass them to the script.
	3.	Write the Script: Use the following C# script inside the Script Task.

Here’s an example of how you might write the script:

using System;
using System.IO;
using System.Collections.Generic;
using Microsoft.SqlServer.Dts.Runtime;

public void Main()
{
    // Retrieve variables from SSIS
    string targetFolder = Dts.Variables["User::FolderPath"].Value.ToString();
    string prefix = Dts.Variables["User::Prefix"].Value.ToString();
    string firstExtension = Dts.Variables["User::FirstExtension"].Value.ToString();
    string secondExtension = Dts.Variables["User::SecondExtension"].Value.ToString();

    // Pattern to match files
    string pattern = $"{prefix}_\\d{{14}}.{firstExtension}";

    // List to hold matching files
    List<string> matchingFiles = new List<string>();

    // Check each file in the folder
    foreach (string filePath in Directory.GetFiles(targetFolder))
    {
        string fileName = Path.GetFileName(filePath);

        // Check if file matches the pattern and has correct extensions
        if (System.Text.RegularExpressions.Regex.IsMatch(fileName, pattern))
        {
            if (fileName.EndsWith($".{firstExtension}") || fileName.EndsWith($".{firstExtension}.{secondExtension}"))
            {
                matchingFiles.Add(filePath);
            }
        }
    }

    // Store results in SSIS variables
    if (matchingFiles.Count > 0)
    {
        Dts.Variables["User::FullFileName"].Value = matchingFiles[0];
        Dts.Variables["User::FileNameWithFirstExtension"].Value = Path.GetFileNameWithoutExtension(matchingFiles[0]);
    }

    // Successful execution
    Dts.TaskResult = (int)ScriptResults.Success;
}

Points to Note:

	•	Regex for Numeric Strings: The script uses a regular expression to match a string of exactly 14 digits (\\d{14}). Make sure your SSIS project references System.Text.RegularExpressions.
	•	File Extensions: The script checks if the file ends with the first extension or the first followed by the second extension.
	•	Error Handling: Add error handling as needed, especially around file access and variable retrieval.
	•	Testing: Test the script with different file names to ensure all conditions are correctly evaluated.

Deploy this script in your SSIS package, configure the variables, and run the package to see if it identifies and processes files as expected.

To handle the scenario where no files match your criteria and to subsequently terminate the SSIS package gracefully with an error, you can add error handling logic to the script in your Script Task. Here’s how you can modify the previous script to achieve this:

	1.	Check for an Empty List: After the file search, check if the list of matching files is empty.
	2.	Raise an Error: If no files are found, raise an error that can be logged and used to stop the package.

Here’s the revised C# script with error handling for no matching files:

using System;
using System.IO;
using System.Collections.Generic;
using Microsoft.SqlServer.Dts.Runtime;

public void Main()
{
    // Retrieve variables from SSIS
    string targetFolder = Dts.Variables["User::FolderPath"].Value.ToString();
    string prefix = Dts.Variables["User::Prefix"].Value.ToString();
    string firstExtension = Dts.Variables["User::FirstExtension"].Value.ToString();
    string secondExtension = Dts.Variables["User::SecondExtension"].Value.ToString();

    // Pattern to match files
    string pattern = $"{prefix}_\\d{{14}}.{firstExtension}";

    // List to hold matching files
    List<string> matchingFiles = new List<string>();

    // Check each file in the folder
    foreach (string filePath in Directory.GetFiles(targetFolder))
    {
        string fileName = Path.GetFileName(filePath);

        // Check if file matches the pattern and has correct extensions
        if (System.Text.RegularExpressions.Regex.IsMatch(fileName, pattern))
        {
            if (fileName.EndsWith($".{firstExtension}") || fileName.EndsWith($".{firstExtension}.{secondExtension}"))
            {
                matchingFiles.Add(filePath);
            }
        }
    }

    // Handle no matching files found
    if (matchingFiles.Count == 0)
    {
        // Raise an error and terminate the package
        Dts.Events.FireError(0, "No Matching Files", "No files matching the specified criteria were found.", "", 0);
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
    else
    {
        // Store results in SSIS variables if files are found
        Dts.Variables["User::FullFileName"].Value = matchingFiles[0];
        Dts.Variables["User::FileNameWithFirstExtension"].Value = Path.GetFileNameWithoutExtension(matchingFiles[0]);
        Dts.TaskResult = (int)ScriptResults.Success;
    }
}

Additional Steps:

	•	Configure Event Handlers in SSIS: To handle errors more robustly, configure the OnError event handler in your SSIS package. This can be used to log errors, send notifications, or execute alternative workflows when an error occurs.
	•	Propagate Errors: Ensure that your Script Task’s “FailPackageOnFailure” property is set to True to make the package fail when the script task fails. This helps in stopping the package execution upon encountering an error.
	•	Test the Error Handling: Simulate conditions where no files are found to ensure that the error handling logic works as expected.

This approach ensures be b b that your SSIS package can gracefully handle and respond to situations where no files meet the specified criteria, enhancing the robustness and reliability of your data workflows.

To efficiently leverage the variables created in the Script Task and manage the workflow for the two subsequent Execute Process Tasks (one for handling files with one extension, and another for handling files with both extensions), you can utilize the control flow features in SSIS, specifically using Precedence Constraints combined with Expressions. Here’s how you can set this up:

Step-by-Step Workflow

	1.	Set Output Variables in Script Task: Ensure your Script Task sets two SSIS variables:
	•	User::FullFileName - The full path of the file.
	•	User::FileNameWithFirstExtension - The file name with the primary extension only.
	2.	Configure Precedence Constraints: After the Script Task, add the two Execute Process Tasks:
	•	Execute Process Task 1: For processing files with only the primary extension.
	•	Execute Process Task 2: For processing files with both extensions.
	3.	Set Expressions on Precedence Constraints: Use expressions on the precedence constraints to determine which Execute Process Task should run based on the file name pattern.

Example Expressions for Constraints

	•	Constraint to Execute Process Task 1 (Files with one extension):

@[User::FullFileName].EndsWith("." + @[User::FirstExtension]) && !@[User::FullFileName].Contains("." + @[User::FirstExtension] + ".")


	•	Constraint to Execute Process Task 2 (Files with both extensions):

@[User::FullFileName].EndsWith("." + @[User::FirstExtension] + "." + @[User::SecondExtension])



Steps to Set Up the Constraints

	1.	Right-click on the arrow (precedence constraint) leading from the Script Task to Execute Process Task 1.
	•	Go to the Properties window.
	•	Set Evaluation operation to Expression.
	•	Set Expression to the one given above for Task 1.
	2.	Right-click on the arrow (precedence constraint) leading from the Script Task to Execute Process Task 2.
	•	Follow the same steps, but use the expression for Task 2.

Considerations for Task Configuration

	•	Execute Process Task Setup: In each Execute Process Task, configure the process to be executed based on the file specified in @[User::FullFileName]. Adjust parameters as needed to handle the file appropriately based on its extension criteria.
	•	Error Handling: Make sure to include error handling in the Execute Process Tasks to manage any issues with file processing, such as file access errors or processing failures.
	•	Logging and Monitoring: Implement logging within each Execute Process Task to track which files are processed and capture any output or errors from the processing operations.

This setup ensures that the correct processing task is executed based on the type of file extension detected by your initial Script Task. It maintains clarity and efficiency in your SSIS package’s control flow, leveraging the variables to conditionally control task execution.
To isolate the file with the most recent creation date from the list of files that have valid names and then write the two output variables, you can modify the C# script in the SSIS Script Task. This involves adding logic to compare the creation dates of the files and then updating the variables accordingly. Here’s how to adjust your script:

Step-by-Step Script Modification

	1.	Find the Most Recent File: After gathering the matching files, iterate through them to find the one with the most recent creation date.
	2.	Update SSIS Variables: Set the SSIS variables to hold the full name and the file name with the first extension of the most recent file.

Here is the revised C# script incorporating these steps:

using System;
using System.IO;
using System.Collections.Generic;
using Microsoft.SqlServer.Dts.Runtime;

public void Main()
{
    // Retrieve variables from SSIS
    string targetFolder = Dts.Variables["User::FolderPath"].Value.ToString();
    string prefix = Dts.Variables["User::Prefix"].Value.ToString();
    string firstExtension = Dts.Variables["User::FirstExtension"].Value.ToString();
    string secondExtension = Dts.Variables["User::SecondExtension"].Value.ToString();

    // Pattern to match files
    string pattern = $"{prefix}_\\d{{14}}.{firstExtension}";

    // Dictionary to hold matching files and their creation dates
    Dictionary<string, DateTime> fileCreationDates = new Dictionary<string, DateTime>();

    // Check each file in the folder
    foreach (string filePath in Directory.GetFiles(targetFolder))
    {
        string fileName = Path.GetFileName(filePath);

        // Check if file matches the pattern and has correct extensions
        if (System.Text.RegularExpressions.Regex.IsMatch(fileName, pattern) &&
            (fileName.EndsWith($".{firstExtension}") || fileName.EndsWith($".{firstExtension}.{secondExtension}")))
        {
            fileCreationDates[filePath] = File.GetCreationTime(filePath);
        }
    }

    // Find the file with the most recent creation date
    if (fileCreationDates.Count > 0)
    {
        string mostRecentFile = null;
        DateTime latestDate = DateTime.MinValue;

        foreach (var file in fileCreationDates)
        {
            if (file.Value > latestDate)
            {
                mostRecentFile = file.Key;
                latestDate = file.Value;
            }
        }

        // Store results in SSIS variables
        Dts.Variables["User::FullFileName"].Value = mostRecentFile;
        Dts.Variables["User::FileNameWithFirstExtension"].Value = Path.GetFileNameWithoutExtension(mostRecentFile);
        Dts.TaskResult = (int)ScriptResults.Success;
    }
    else
    {
        // Raise an error and terminate the package if no files found
        Dts.Events.FireError(0, "No Matching Files", "No files matching the specified criteria were found.", "", 0);
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}

Key Points:

	•	File Creation Dates: The script now uses a Dictionary to store file paths and their creation dates, making it easier to find the most recent file.
	•	Most Recent File Selection: The script iterates through the dictionary to determine the file with the most recent creation date.
	•	Error Handling and Package Termination: If no files match the criteria, the script fires an error and sets the task result to Failure, which can be used to terminate the package gracefully.

With this script, your SSIS package will correctly identify the most recently created file that matches your criteria and populate the appropriate variables for subsequent processing steps. This approach ensures that your data workflows are both accurate and efficient.



