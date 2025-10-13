---
layout      : single
title       : SharePoint CSOM Lambda expressions in PowerShell
summary     : SharePoint CSOM Lambda expressions in PowerShell
categories  : SharePoint
tags        : [SharePoint, SharePoint Online, CSOM, Lambda expressions, PowerShell]
date        : 2025-10-12 18:28:00
commentId   : 4
permalink   : /SharePointCSOMLambdaExpressionsInPowerShell
toc         : true
classes     : wide
toc_icon    : "cog"
toc_label   : "My Table of Contents"
---

# Summary

Using the SharePoint CSOM API with lambda expressions from PowerShell can be tricky. Gary Lapointe's blog post is a good reference: https://blog.falchionconsulting.com/2015/03/loading-specific-values-using-lambda-expressions-and-the-sharepoint-csom-api-with-windows-powershell/

This article explains an improved PowerShell helper (Load-CSOMProperties) that makes it straightforward to request nested/child properties (for example, RoleAssignments -> Member and RoleDefinitionBindings) in a single server call. The goal is to reduce round-trips and simplify code that loads collection children via CSOM.

# Why this is useful

- CSOM loads are explicit: to avoid extra server calls you must request exactly the properties and child collections you need.
- Expressing nested includes (collections and their child fields) with lambda expressions in PowerShell is cumbersome because of generics and expression-tree APIs.
- The improved Load-CSOMProperties function builds the required expression trees and supports:
  - Loading properties for a ClientObject or ClientObjectCollection
  - Expanding a child collection and specifying multiple child properties to include (e.g., RoleAssignments.Member and RoleAssignments.RoleDefinitionBindings)

# Example C# (for context)

The C# snippet below shows the target behavior: load an item's RoleAssignments including Member and RoleDefinitionBindings in one request.

```C#
var items = library.GetItems(CamlQuery.CreateAllItemsQuery());
clientContext.Load(items, a => a.Include(
    y => y.FileSystemObjectType,
    y => y.Id,
    y => y.HasUniqueRoleAssignments,
    y => y.RoleAssignments.Include(
        z => z.Member,
        z => z.RoleDefinitionBindings)));
clientContext.ExecuteQuery();

```

# Improved Load-CSOMProperties (PowerShell)

The improved Load-CSOMProperties function below builds expression trees to support nested child collection expansion and multiple inner properties. It includes caching for reflection- and expression-related operations to keep repeated calls efficient.

```powershell
$script:__IncludeMethodCache = @{}
$script:__LambdaGenericCache = @{}

function Get-IncludeGeneric {
    param([Type]$ElementType)
    $key = $ElementType.AssemblyQualifiedName
    if ($script:__IncludeMethodCache.ContainsKey($key)) { return $script:__IncludeMethodCache[$key] }
    $includeMethod = [Microsoft.SharePoint.Client.ClientObjectQueryableExtension].GetMethods() |
        Where-Object { $_.Name -eq 'Include' -and $_.IsGenericMethodDefinition -and $_.GetParameters().Length -eq 2 } |
        Select-Object -First 1
    $gen = $includeMethod.MakeGenericMethod($ElementType)
    $script:__IncludeMethodCache[$key] = $gen
    return $gen
}

function Get-LambdaGeneric {
    param([Type]$FromType)
    $key = $FromType.AssemblyQualifiedName
    if ($script:__LambdaGenericCache.ContainsKey($key)) { return $script:__LambdaGenericCache[$key] }
    $exprType = [System.Linq.Expressions.Expression]
    $parameterExprType = [System.Linq.Expressions.ParameterExpression].MakeArrayType()
    $lambdaMethod = $exprType.GetMethods() |
        Where-Object { $_.Name -eq "Lambda" -and $_.IsGenericMethod -and $_.GetParameters().Length -eq 2 -and $_.GetParameters()[1].ParameterType -eq $parameterExprType } |
        Select-Object -First 1
    $generic = Invoke-Expression "`$lambdaMethod.MakeGenericMethod([System.Func``2[$($FromType.FullName),System.Object]])"
    $script:__LambdaGenericCache[$key] = $generic
    return $generic
}

# Cache for property infos (type + property name)
if (-not $script:__PropertyInfoCache) { $script:__PropertyInfoCache = @{} }
function Get-PropertyInfoCached {
    param(
        [Parameter(Mandatory)][Type]$Type,
        [Parameter(Mandatory)][string]$Name
    )
    $key = $Type.AssemblyQualifiedName + '|' + $Name
    if ($script:__PropertyInfoCache.ContainsKey($key)) {
        return $script:__PropertyInfoCache[$key]
    }
    $binding = [System.Reflection.BindingFlags]::Instance -bor [System.Reflection.BindingFlags]::Public -bor [System.Reflection.BindingFlags]::FlattenHierarchy
    $pi = $Type.GetProperty($Name, $binding)
    $script:__PropertyInfoCache[$key] = $pi
    return $pi
}

function global:Load-CSOMProperties {
    [CmdletBinding(DefaultParameterSetName = 'ClientObject')]
    param (
        # The Microsoft.SharePoint.Client.ClientObject to populate.
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, Position = 0, ParameterSetName = "ClientObject")]
        [Microsoft.SharePoint.Client.ClientObject]
        $object,

        # The Microsoft.SharePoint.Client.ClientObject that contains the collection object.
        [Parameter(Mandatory = $false, ValueFromPipeline = $true, Position = 0, ParameterSetName = "ClientObjectCollection")]
        [Microsoft.SharePoint.Client.ClientObject]
        $parentObject,

        # The Microsoft.SharePoint.Client.ClientObjectCollection to populate.
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, Position = 1, ParameterSetName = "ClientObjectCollection")]
        [Microsoft.SharePoint.Client.ClientObjectCollection]
        $collectionObject,

        # The object properties to populate
        [Parameter(Mandatory = $true, Position = 1, ParameterSetName = "ClientObject")]
        [Parameter(Mandatory = $true, Position = 2, ParameterSetName = "ClientObjectCollection")]
        [string[]]
        $propertyNames,

        # The parent object's property name corresponding to the collection object to retrieve (this is required to build the correct lamda expression).
        [Parameter(Mandatory = $false, Position = 3, ParameterSetName = "ClientObjectCollection")]
        [string]
        $parentPropertyName,

        # If specified, execute the ClientContext.ExecuteQuery() method.
        [Parameter(Mandatory = $false, Position = 4)]
        [switch]
        $executeQuery,

        [Parameter(Mandatory = $false, Position = 5)]
        [switch]
        $expandAdditionalProperty,
        
        [Parameter(Mandatory = $false, Position = 6)]
        [string]
        $expandAdditionalPropertyName,

        [Parameter(Mandatory = $false, Position = 7)]
        [string[]]
        $expandAdditionalPropertyNames
    )

    begin { }
    process {
        # Resolve element type
        if ($PsCmdlet.ParameterSetName -eq "ClientObject") {
            $type = $object.GetType()
        } else {
            $type = $collectionObject.GetType()
            if ($collectionObject -is [Microsoft.SharePoint.Client.ClientObjectCollection]) {
                $type = $collectionObject.GetType().BaseType.GenericTypeArguments[0]
            }
        }

        # Get cached main lambda generic
        $lambdaMethodGeneric = Get-LambdaGeneric -FromType $type
        $expressions = @()

        foreach ($propertyName in $propertyNames) {
            $param1 = [System.Linq.Expressions.Expression]::Parameter($type, "p")
            try {
                if ($expandAdditionalProperty -and $propertyName -eq $expandAdditionalPropertyName) {
                    # Inherited / collection expansion (e.g. RoleAssignments)
                    $propInfo = Get-PropertyInfoCached -Type $type -Name $expandAdditionalPropertyName
                    if (-not $propInfo) {
                        Write-error "Expand property '$expandAdditionalPropertyName' not found on $type (or bases)"
                        return
                    }
                    $collectionPropExpr = [System.Linq.Expressions.Expression]::Property($param1, $expandAdditionalPropertyName)
                    $collectionPropType = $propInfo.PropertyType

                    # Derive element type of ClientObjectCollection<T>
                    $elementType = $null
                    if ($collectionPropType.BaseType -and $collectionPropType.BaseType.IsGenericType) {
                        $elementType = $collectionPropType.BaseType.GetGenericArguments()[0]
                    }
                    if (-not $elementType) {
                        Write-Error "Cannot derive element type for '$expandAdditionalPropertyName' on $type" 
                        return
                    }

                    # Build inner lambdas with cached generic
                    $lambdaMethodGenericInner = Get-LambdaGeneric -FromType $elementType
                    $innerParam = [System.Linq.Expressions.Expression]::Parameter($elementType, "r")
                    $innerQuoted = @()

                    foreach ($innerPropName in $expandAdditionalPropertyNames) {
                        try {
                            $innerPropExpr = [System.Linq.Expressions.Expression]::Property($innerParam, $innerPropName)
                        } catch {
                            Write-Warning "Skip missing inner property '$innerPropName' on $elementType"
                            continue
                        }
                        $innerBody   = [System.Linq.Expressions.Expression]::Convert($innerPropExpr, [System.Object])
                        $innerLambda = $lambdaMethodGenericInner.Invoke($null, @($innerBody, [System.Linq.Expressions.ParameterExpression[]]@($innerParam)))
                        $innerQuoted += [System.Linq.Expressions.Expression]::Quote($innerLambda)
                    }

                    if ($innerQuoted.Count -eq 0) {
                        Write-Warning  "No valid inner properties for '$expandAdditionalPropertyName'; loading raw collection only."
                        $name1 = $collectionPropExpr
                    } else {
                        $includeGeneric = Get-IncludeGeneric -ElementType $elementType
                        $lambdaElemInnerType = $innerQuoted[0].Type
                        $innerArrayExpr = [System.Linq.Expressions.Expression]::NewArrayInit($lambdaElemInnerType, $innerQuoted)
                        $name1 = [System.Linq.Expressions.Expression]::Call($null, $includeGeneric, @($collectionPropExpr, $innerArrayExpr))
                    }
                } else {
                    $name1 = [System.Linq.Expressions.Expression]::Property($param1, $propertyName)
                }
            } catch {
                Write-Error "Instance property '$propertyName' is not defined for type $type"
                return
            }

            $body1       = [System.Linq.Expressions.Expression]::Convert($name1, [System.Object])
            $expression1 = $lambdaMethodGeneric.Invoke($null, [object[]]@($body1, [System.Linq.Expressions.ParameterExpression[]]@($param1)))

            if ($PsCmdlet.ParameterSetName -eq "ClientObjectCollection") {
                $expression1 = [System.Linq.Expressions.Expression]::Quote($expression1)
            }
            $expressions += $expression1
        }

        if (-not $expressions -or $expressions.Count -eq 0) {
            Write-Warning "No expressions generated for type $type; skipping load."
            return
        }

        if ($PsCmdlet.ParameterSetName -eq "ClientObject") {
            $object.Context.Load($object, $expressions)
            if ($executeQuery) { $object.Context.ExecuteQuery() }
        }
        elseif ($null -eq $parentObject) {
            # Collection include
            $listItems      = $collectionObject
            $ctx            = $listItems.Context
            $collectionType = $listItems.GetType()
            $itemsParam     = [System.Linq.Expressions.Expression]::Parameter($collectionType, 'items')

            $lambdaElemType  = $expressions[0].Type
            $retrievalsArray = [System.Linq.Expressions.Expression]::NewArrayInit($lambdaElemType, $expressions)

            $includeGeneric = Get-IncludeGeneric -ElementType $type
            $callInclude    = [System.Linq.Expressions.Expression]::Call($null, $includeGeneric, @($itemsParam, $retrievalsArray))

            $outerLambdaGeneric = Get-LambdaGeneric -FromType $collectionType
            $outerLambda        = $outerLambdaGeneric.Invoke($null, @($callInclude, [System.Linq.Expressions.ParameterExpression[]]@($itemsParam)))

            $ctx.Load($listItems, $outerLambda)
            if ($executeQuery) { $ctx.ExecuteQuery() }
        }
        else {
            # Parent + child collection include
            $parentType     = $parentObject.GetType()
            $newArrayInitParam1 = Invoke-Expression "[System.Linq.Expressions.Expression``1[System.Func````2[$($type.FullName),System.Object]]]"
            $newArrayInit   = [System.Linq.Expressions.Expression]::NewArrayInit($newArrayInitParam1, $expressions)

            $collectionParam    = [System.Linq.Expressions.Expression]::Parameter($parentType, "cp")
            $collectionProperty = [System.Linq.Expressions.Expression]::Property($collectionParam, $parentPropertyName)

            $expressionArray    = @($collectionProperty, $newArrayInit)
            $includeGeneric     = Get-IncludeGeneric -ElementType $type
            $callMethod         = [System.Linq.Expressions.Expression]::Call($null, $includeGeneric, $expressionArray)

            $lambdaMethodGeneric2 = Get-LambdaGeneric -FromType $parentType
            $expression2          = $lambdaMethodGeneric2.Invoke($null, @($callMethod, [System.Linq.Expressions.ParameterExpression[]]@($collectionParam)))

            $parentObject.Context.Load($parentObject, $expression2)
            if ($executeQuery) { $parentObject.Context.ExecuteQuery() }
        }
    }
    end { }
}

```

# Usage example

Below is a PowerShell usage example that retrieves list items and expands RoleAssignments in a single request.

```PowerShell
$listItems = $list.GetItems([Microsoft.SharePoint.Client.CamlQuery]::CreateAllItemsQuery())

Load-CSOMProperties -collectionObject $listItems `
    -propertyNames @("FileSystemObjectType", "Id", "HasUniqueRoleAssignments", "RoleAssignments") `
    -expandAdditionalProperty -expandAdditionalPropertyName "RoleAssignments" `
    -expandAdditionalPropertyNames @("Member", "RoleDefinitionBindings")
$context.ExecuteQuery()
```

# References

- Original inspiration / reference: Gary Lapointe — Loading specific values using lambda expressions and the SharePoint CSOM API with Windows PowerShell
  https://blog.falchionconsulting.com/2015/03/loading-specific-values-using-lambda-expressions-and-the-sharepoint-csom-api-with-windows-powershell/
