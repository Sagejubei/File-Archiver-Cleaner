# File-Archiver-Cleaner
# Script: Automated File Archiver and Cleaner
# Set the error action preference to 'Stop'
# This will cause non-terminating errors to become terminating errors
$ErrorActionPreference = 'Stop'

# Define paths
$sourceFolder = "C:\Users\vcabrera\Desktop"
$archiveFolder = "C:\Users\vcabrera\Desktop\Archives"

# Define alternate archive location for handling permission issues
$alternateArchiveFolder = "C:\Users\vcabrera\Desktop\AlternateArchives"

# Ensure alternate archive folder exists
if (!(Test-Path $alternateArchiveFolder)) {
    try {
        New-Item -ItemType Directory -Path $alternateArchiveFolder
        Write-Host "Created alternate archive folder: $alternateArchiveFolder"
    }
    catch {
        $errorMessage = "failed to create alternate archive folder: $alternateArchiveFolder"
        Write-Error $errorMessage
        Log-Error $errorMessage
        exit
    }
}

# Check if source folder exists
if (!(Test-Path $sourceFolder)) {
    Write-Host "Error: Source folder does not exist: $sourceFolder"
    exit
}

# Ensure archive folder exists
if (!(Test-Path $archiveFolder)) {
    New-Item -ItemType Directory -Path $archiveFolder #-ErrorAction Stop
    Write-Host "Created archive folder: $archiveFolder"
}

# Function to log errors to a file
function Log-Error {
    param(
        [string]$message 
    )
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "$timestamp - $message"
    Add-Content -Path "Desktop\FileArchiverErrors.log" -Value $logMMessage
}

# Get files older than 30 days
$oldFiles = [System.Collections.ArrayList]::new()
$cutOffDate = (Get-Date).AddDays(-30)
    foreach ($file in Get-ChildItem -Path $sourceFolder -File -ErrorAction SilentlyContinue) {
        if ($file.LastWriteTime -lt $cutOffDate) {
            [void]$oldFiles.Add($file)
    }
}

# Process each file
foreach ($file in $oldFiles) {
    try {
        # Attempt to compress the file
        Compress-Archive -Path $file.FullName -DestinationPath "$archiveFolder\$($file.BaseName).zip" -ErrorAction Stop
        Write-Host "Compressed file: $(file.Name)"
    }
    catch [System.UnauthorizedAccessException]{
        Write-Error "Access denied to primary archive folder. Attempting to use alternate location."
        try {
            Compress-Archvive -Path $file.FullName -DestinationPath "$alternateArchiveFolder\$($file.BaseName).zip"
            -ErrorAction Stop
            Write-Host "Compressed file to alternate lcoation: $($file.Name)"
        }
        catch {
            $errorMessage = "Failed to compress file to alternate location: $($file.Name). Error: $($_.Exception.Message)"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
    }   
    catch  {
        # If compression fails for any other reason, attempt to copy the file
        $errorMessage = "Failed to compress file: $($FILE.nAME). Attempting to copy to the archive folder."
        Write-Error $errorMessageLog-Error $errorMessage
        try{
            Copy-Item -Path $file.FullName -Destination $archiveFolder -ErrorAction Stop
            Write-Host "Copied file to the archive folder: $($file.Name)"
        }
        catch [System.UnauthorizedAccessException] {
            Write-Host "Access denied to primary archive folder. Attempting to copy to alternate location."
            try {
                Copy-Item -Path $file.FullName -Destination $alternateArchiveFolder -ErrorAction Stop
                Write-Host "COpied file to alternate archive folder: $($file.Name)"
            }
            catch {
                $errorMessage = "failed to copy file to alternate location: $($file.Name). Error: $($_.Exception.Message)"
                Write-Error $errorMessage
                Log-Error $errorMessage
            }
        }
        catch {
            $errorMessage = "Failed to copy: $($file.Name). Error: $($_.Exception.Message)"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
    }
    finally {
        # Attempt to delete the original file, regardless of whether it was compressed or copied
        try {
            Remove-Item -Path $filr.FullName -ErrorAction Stop
            Write-Host "Deleted original file: $($file.Nmae)"
        }
        catch [System.UnauthorizedAccessException] {
            $errorMessage = "Access denied when trying to delete original file: $($file.Name)"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
        catch {
            $errorMessage = "Failed to delete original file: $($file.Name). Error: $($_.Exception.Message)"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
    }
}

# Clean up old archive files (older than 1 year)
$oldArchivesCount = 0
$cutOffDate = (Get-Date).AddYears(-1)
$allFiles = Get-ChildItem -Path $archiveFolder -File -ErrorAction SilentlyContinue

foreach ($file in $allFiles) {
   if ($file.LastWriteTime -lt $cutOffDate) {
        try {
            Remove-Item -Path $file.FullName -ErrorAction Stop
            $oldArchivsCount++
            Write-Host "Removed old archiove: $file"
        }
        catch [System.UnauthorizedAccessException] {
            $errorMessage = "Access denied when trying to remove old archive: $file"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
        catch {
            $errorMessage = "Failed to remove old archive: $file. Error: $($_.Exception.Message)"
            Write-Error $errorMessage
            Log-Error $errorMessage
        }
   }
}

# Generate and display summary
$summary = "File Archiving and Cleaning Summary:`n"
$summary += "------------------------------------`n"
$summary += "Files moved to archive: {0}`n" -f $oldFiles.Count
$summary += "Old archives removed: {0}" -f $oldArchivesCount
Write-Host $summary

# Prompt user for file list viewing if we processed any files
if ($oldFiles.Count -gt 0 -or $oldArchivesCount -gt 0) {
    if (Read-Host "Do you want to view the list of processed files? (Y/N)" -eq 'Y') {
        $oldFiles | Select-Object Name, LastWriteTime
    }
}
