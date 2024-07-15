# Test
Test


# Import CSV files
$B = Import-Csv -Path "B.csv"
$C = Import-Csv -Path "C.csv"

# Add unique record IDs
$i = 1
$B | ForEach-Object {
    $_ | Add-Member -MemberType NoteProperty -Name "b0" -Value $i -Force
    $i++
}
$j = 1
$C | ForEach-Object {
    $_ | Add-Member -MemberType NoteProperty -Name "c0" -Value $j -Force
    $j++
}

# Filter B.csv to only leave records containing the substring "5TEST" in column b1
$filteredB = $B | Where-Object { $_.b1 -like "*5TEST*" }

# Save the filtered B.csv records for debugging
$filteredB | Export-Csv -Path "debug_B_filtered.csv" -NoTypeInformation

# Create a list of all unique string values found in column b6
$uniqueB6Values = $filteredB | Select-Object -ExpandProperty b6 | Sort-Object -Unique

# Save the unique b6 values for debugging
$uniqueB6Values | ForEach-Object { [PSCustomObject]@{ b6 = $_ } } | Export-Csv -Path "debug_B_b6_list.csv" -NoTypeInformation

# Filter C.csv to only leave records containing a value in c2 that is in the list of uniqueB6Values
$filteredC = $C | Where-Object { $uniqueB6Values -contains $_.c2 }

# Save the filtered C.csv records for debugging
$filteredC | Export-Csv -Path "debug_C_filtered.csv" -NoTypeInformation

# Define the join key columns
$keyColumnB = "b6"
$keyColumnC = "c2"

# Create a hashtable to store combined rows for the join
$joinedRows = @()

# Create a dictionary for quick lookup of filteredC rows by key
$C_dict = @{}
foreach ($row in $filteredC) {
    $key = $row.$keyColumnC
    if (![string]::IsNullOrEmpty($key)) {
        if (-not $C_dict.ContainsKey($key)) {
            $C_dict[$key] = @()
        }
        $C_dict[$key] += $row
    }
}

# Process filteredB and join with corresponding entries in filteredC
foreach ($rowB in $filteredB) {
    $keyB = $rowB.$keyColumnB
    if (![string]::IsNullOrEmpty($keyB) -and $C_dict.ContainsKey($keyB)) {
        foreach ($rowC in $C_dict[$keyB]) {
            # Create a combined row
            $combinedRow = New-Object PSObject

            # Add columns from B.csv
            foreach ($column in $rowB.PSObject.Properties.Name) {
                Add-Member -InputObject $combinedRow -MemberType NoteProperty -Name $column -Value $rowB.$column -Force
            }

            # Add columns from C.csv
            foreach ($column in $rowC.PSObject.Properties.Name) {
                Add-Member -InputObject $combinedRow -MemberType NoteProperty -Name $column -Value $rowC.$column -Force
            }

            # Add the combined row to the joined rows
            $joinedRows += $combinedRow
        }
    }
}

# Create an ordered array of column names for the output CSV
$columnOrder = @("b0") + $B[0].PSObject.Properties.Name + @("c0") + $C[0].PSObject.Properties.Name

# Check if BC.csv exists and remove it if necessary
$bcPath = "BC.csv"
if (Test-Path $bcPath) {
    Remove-Item $bcPath
}

# Export the result to BC.csv
$joinedRows | Export-Csv -Path $bcPath -NoTypeInformation
