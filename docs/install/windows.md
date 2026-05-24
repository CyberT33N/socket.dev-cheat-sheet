Die belastbare Aussage ist: `sfw` ist nicht nur fuer `install` gedacht. Dokumentiert ist die generische Form `sfw <command>...`, und die dokumentierten Package-Manager-Entry-Points sind `npm`, `yarn`, `pnpm`, `pip`, `uv` und `cargo` ([Socket Firewall Free](https://docs.socket.dev/docs/socket-firewall-free)). Genau deshalb ist die architekturell saubere Loesung, die **Kommandonamen** zu wrappen statt eine fragile Liste einzelner Subcommands zu pflegen. `py -m pip` ist in der Doku **nicht** als `sfw`-Entry-Point genannt, also habe ich es bewusst **nicht** automatisch umgebogen. Der `sfw-installer` ist dabei nur ein Release-/Binary-Wrapper, keine zusaetzliche Command-Surface ([sfw-installer README](https://github.com/SocketDev/sfw-installer), [v1.10.0 Release](https://github.com/SocketDev/sfw-free/releases/tag/v1.10.0)).

`⚠️ ANNAHME:` Die konkrete Signatur von `MPMI-Global` ist in deinen Quellen nicht oeffentlich dokumentiert. Das Skript nutzt deshalb standardmaessig `MPMI-Global sfw`. Falls dein lokaler Vertrag anders aussieht, passe nur `-MpmiArguments` an. Geprueft habe ich das Skript in `powershell.exe` mit `-WhatIf`; dabei wurde die vorhandene `sfw`-Version korrekt ueber die Versionsausgabe erkannt und nur der Profilblock als geplante Aenderung ausgewiesen.

```powershell
<#
.SYNOPSIS
Ensures Socket Firewall Free is installed and wires profile wrappers.
.DESCRIPTION
Checks whether `sfw` is installed by evaluating `sfw --version`. If no version is
available, the script invokes the global `MPMI-Global` command to install SWF.
Afterwards it verifies that the PowerShell profile contains a managed wrapper block
for all documented SWF package-manager entry points: npm, yarn, pnpm, pip, uv,
and cargo. The wrapper block is added only once and logs every proxied call with
an explicit `swf inception` message.
.NOTES
Adjust -MpmiArguments if your local MPMI-Global contract expects a different
argument shape than the default package-name form.
#>
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
param(
    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string]$ProfilePath = $PROFILE.CurrentUserCurrentHost,

    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string]$SwfCommandName = 'sfw',

    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string]$MpmiCommandName = 'MPMI-Global',

    [Parameter()]
    [ValidateNotNullOrEmpty()]
    [string[]]$MpmiArguments = @('sfw')
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$profileBlockStart = '# >>> swf inception wrapper >>>'
$profileBlockEnd = '# <<< swf inception wrapper <<<'
$managedBlockRequiredSnippets = @(
    'function Invoke-SfwInception',
    'function npm',
    'function yarn',
    'function pnpm',
    'function pip',
    'function uv',
    'function cargo'
)

function Write-SetupLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateSet('INFO', 'WARN', 'SUCCESS')]
        [string]$Level,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$Message
    )

    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    Write-Information ('[{0}] [SWF-SETUP] [{1}] {2}' -f $timestamp, $Level, $Message) -InformationAction Continue
}

function Update-SessionPathFromEnvironment {
    [CmdletBinding()]
    param()

    $machinePath = [System.Environment]::GetEnvironmentVariable('Path', 'Machine')
    $userPath = [System.Environment]::GetEnvironmentVariable('Path', 'User')

    $pathEntries = @($machinePath, $userPath) | Where-Object {
        -not [string]::IsNullOrWhiteSpace($_)
    }

    if ($pathEntries.Count -gt 0) {
        $env:Path = $pathEntries -join ';'
    }
}

function Get-SfwVersionInfo {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$CommandName
    )

    $sfwCommand = Get-Command -Name $CommandName -ErrorAction SilentlyContinue
    if (-not $sfwCommand) {
        return $null
    }

    try {
        $versionOutput = & $CommandName --version 2>$null
        if ($LASTEXITCODE -ne 0) {
            return $null
        }

        $versionText = (($versionOutput | ForEach-Object { $_.ToString() }) -join [System.Environment]::NewLine).Trim()
        if ([string]::IsNullOrWhiteSpace($versionText)) {
            return $null
        }

        $versionMatch = [System.Text.RegularExpressions.Regex]::Match(
            $versionText,
            'version\s+(?<Version>\d+(?:\.\d+){1,3}(?:[-+][0-9A-Za-z\.-]+)?)',
            [System.Text.RegularExpressions.RegexOptions]::IgnoreCase
        )

        if (-not $versionMatch.Success) {
            return $null
        }

        [pscustomobject]@{
            Command   = $CommandName
            Version   = $versionMatch.Groups['Version'].Value
            RawOutput = $versionText
        }
    }
    catch {
        $null
    }
}

function Assert-CommandExists {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$CommandName
    )

    $null = Get-Command -Name $CommandName -ErrorAction Stop
}

function Get-ProfileContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$Path
    )

    if (-not (Test-Path -LiteralPath $Path -PathType Leaf)) {
        return ''
    }

    $content = Get-Content -LiteralPath $Path -Raw -ErrorAction Stop
    if ($null -eq $content) {
        return ''
    }

    return $content
}

function Get-SfwProfileBlock {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$BlockStart,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$BlockEnd
    )

    $template = @'
__BLOCK_START__
function Write-SfwInceptionLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$Message
    )

    Write-Information "[swf inception] $Message" -InformationAction Continue
}

function Format-SfwInceptionCommandLine {
    [CmdletBinding()]
    param(
        [Parameter()]
        [AllowEmptyCollection()]
        [object[]]$Arguments = @()
    )

    $tokens = foreach ($argument in $Arguments) {
        if ($null -eq $argument) {
            '""'
            continue
        }

        $text = [string]$argument
        if ($text -match '\s') {
            '"' + $text + '"'
        }
        else {
            $text
        }
    }

    $tokens -join ' '
}

function Invoke-SfwInception {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateSet('npm', 'yarn', 'pnpm', 'pip', 'uv', 'cargo')]
        [string]$PackageManager,

        [Parameter()]
        [AllowEmptyCollection()]
        [object[]]$Arguments = @()
    )

    $sfwCommand = Get-Command -Name 'sfw' -ErrorAction SilentlyContinue
    if (-not $sfwCommand) {
        throw "The 'sfw' command was not found in PATH. Run the SWF bootstrap script first."
    }

    $forwardedArguments = @($PackageManager) + @($Arguments)
    $displayCommandLine = Format-SfwInceptionCommandLine -Arguments $forwardedArguments

    Write-SfwInceptionLog -Message ("Proxying package manager command through Socket Firewall Free: sfw {0}" -f $displayCommandLine)

    & 'sfw' @forwardedArguments
}

function npm {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'npm' -Arguments $Arguments
}

function yarn {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'yarn' -Arguments $Arguments
}

function pnpm {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'pnpm' -Arguments $Arguments
}

function pip {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'pip' -Arguments $Arguments
}

function uv {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'uv' -Arguments $Arguments
}

function cargo {
    [CmdletBinding(PositionalBinding = $false)]
    param(
        [Parameter(ValueFromRemainingArguments = $true)]
        [AllowEmptyCollection()]
        [object[]]$Arguments
    )

    Invoke-SfwInception -PackageManager 'cargo' -Arguments $Arguments
}
__BLOCK_END__
'@

    return $template.Replace('__BLOCK_START__', $BlockStart).Replace('__BLOCK_END__', $BlockEnd)
}

function Test-SfwProfileBlockPresent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]$ProfileContent,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$BlockStart,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$BlockEnd,

        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string[]]$RequiredSnippets
    )

    $startPresent = $ProfileContent.Contains($BlockStart)
    $endPresent = $ProfileContent.Contains($BlockEnd)

    if ($startPresent -xor $endPresent) {
        throw 'Detected a partial SWF-managed profile block. Remove the incomplete block and re-run the script.'
    }

    if (-not ($startPresent -and $endPresent)) {
        return $false
    }

    $startIndex = $ProfileContent.IndexOf($BlockStart, [System.StringComparison]::Ordinal)
    $endIndex = $ProfileContent.IndexOf($BlockEnd, [System.StringComparison]::Ordinal)

    if ($endIndex -lt $startIndex) {
        throw 'Detected an invalid SWF-managed profile block ordering. Remove the existing block and re-run the script.'
    }

    $blockLength = ($endIndex + $BlockEnd.Length) - $startIndex
    $existingBlock = $ProfileContent.Substring($startIndex, $blockLength)

    foreach ($snippet in $RequiredSnippets) {
        if ($existingBlock.IndexOf($snippet, [System.StringComparison]::OrdinalIgnoreCase) -lt 0) {
            throw 'Detected an incomplete or manually modified SWF-managed profile block. Remove the existing block and re-run the script.'
        }
    }

    return $true
}

function Ensure-ProfileParentDirectory {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$Path
    )

    $directoryPath = Split-Path -Path $Path -Parent
    if ([string]::IsNullOrWhiteSpace($directoryPath)) {
        return
    }

    if (-not (Test-Path -LiteralPath $directoryPath -PathType Container)) {
        New-Item -Path $directoryPath -ItemType Directory -Force -ErrorAction Stop | Out-Null
    }
}

Write-SetupLog -Level 'INFO' -Message 'Starting SWF bootstrap and profile wrapper validation.'
Write-SetupLog -Level 'INFO' -Message 'Documented SWF package-manager entry points: npm, yarn, pnpm, pip, uv, cargo.'
Write-SetupLog -Level 'WARN' -Message "The Windows launcher 'py -m pip' is not a documented sfw entry point and is intentionally not wrapped. Use 'pip ...' or 'uv pip ...' for documented SWF coverage."

$versionInfo = Get-SfwVersionInfo -CommandName $SwfCommandName

if ($null -ne $versionInfo) {
    Write-SetupLog -Level 'SUCCESS' -Message ("Detected SWF installation via version probe: {0}" -f $versionInfo.Version)
}
else {
    Write-SetupLog -Level 'WARN' -Message ('No SWF version could be resolved from ''{0} --version''. Installation is required.' -f $SwfCommandName)

    if ($PSCmdlet.ShouldProcess($SwfCommandName, ('Install SWF via {0} {1}' -f $MpmiCommandName, ($MpmiArguments -join ' ')))) {
        Assert-CommandExists -CommandName $MpmiCommandName
        Write-SetupLog -Level 'INFO' -Message ('Invoking installation command: {0} {1}' -f $MpmiCommandName, ($MpmiArguments -join ' '))

        & $MpmiCommandName @MpmiArguments

        Update-SessionPathFromEnvironment
        $versionInfo = Get-SfwVersionInfo -CommandName $SwfCommandName
        if ($null -eq $versionInfo) {
            throw ('SWF installation command completed, but ''{0} --version'' still returned no version. Verify the MPMI-Global package name and PATH propagation.' -f $SwfCommandName)
        }

        Write-SetupLog -Level 'SUCCESS' -Message ("SWF installation verified via version probe: {0}" -f $versionInfo.Version)
    }
}

$profileContent = Get-ProfileContent -Path $ProfilePath
$profileBlockPresent = Test-SfwProfileBlockPresent -ProfileContent $profileContent -BlockStart $profileBlockStart -BlockEnd $profileBlockEnd -RequiredSnippets $managedBlockRequiredSnippets

if ($profileBlockPresent) {
    Write-SetupLog -Level 'SUCCESS' -Message ("The managed SWF wrapper block is already present in profile '{0}'. Skipping profile update." -f $ProfilePath)
}
else {
    Write-SetupLog -Level 'WARN' -Message ("The managed SWF wrapper block is missing from profile '{0}'." -f $ProfilePath)

    if ($PSCmdlet.ShouldProcess($ProfilePath, 'Create or update profile with SWF wrapper block')) {
        Ensure-ProfileParentDirectory -Path $ProfilePath

        $profileBlock = Get-SfwProfileBlock -BlockStart $profileBlockStart -BlockEnd $profileBlockEnd
        if ([string]::IsNullOrWhiteSpace($profileContent)) {
            Set-Content -LiteralPath $ProfilePath -Value $profileBlock -Encoding UTF8 -ErrorAction Stop
        }
        else {
            Add-Content -LiteralPath $ProfilePath -Value ([System.Environment]::NewLine + [System.Environment]::NewLine + $profileBlock) -Encoding UTF8 -ErrorAction Stop
        }

        $profileContent = Get-ProfileContent -Path $ProfilePath
        $profileBlockPresent = Test-SfwProfileBlockPresent -ProfileContent $profileContent -BlockStart $profileBlockStart -BlockEnd $profileBlockEnd -RequiredSnippets $managedBlockRequiredSnippets
        if (-not $profileBlockPresent) {
            throw 'Profile update was attempted, but the managed SWF block could not be verified afterwards.'
        }

        Write-SetupLog -Level 'SUCCESS' -Message ("Added the managed SWF wrapper block to profile '{0}'." -f $ProfilePath)
    }
}

Write-SetupLog -Level 'INFO' -Message ("Finished. Start a new shell or run: . '{0}'" -f $ProfilePath)
```

Wenn du die Wrapper sowohl in `powershell.exe` als auch in `pwsh` willst, fuehre das Skript einmal pro Host aus, weil `$PROFILE.CurrentUserCurrentHost` host-spezifisch ist. Wenn dein `MPMI-Global` lokal nicht `MPMI-Global sfw` erwartet, sag mir kurz die exakte Signatur, dann passe ich dir die Standardwerte sofort sauber darauf an.
