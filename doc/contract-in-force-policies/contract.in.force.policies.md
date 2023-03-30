# Contract In Force Policies Specification

This document defines interoperable policies for specifying in force periods for contract agreements. An in force period can be defined as a __duration__ or a __fixed date__.
All dates must be expressed as UTC.

## 1. Duration

A duration is a period of time starting from an offset. Catena-X defines a simple expression language of type `cx:dateExpression` for specifying the offset and duration in time
units:

```<offset> + <numeric value> ms|s|m|h|d```

The following values are supported for `<offset>`:

| Value                  | Description                                                                                                                              |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| contractAgreementStart | The start of the contract agreement defined as the timestamp when the provider enters the FINALIZED state expressed in UTC epoch seconds |

The following values are supported for the time unit:

| Value | Description  |
|-------|--------------|
| ms    | milliseconds |
| s     | seconds      |
| m     | minutes      |
| h     | hours        |
| d     | days         |

A duration is defined in a `ContractDefinition` using the following policy and `cx:inForceDate` operands:

```json
{
  "@context": {
    "cx": "https://w3id.org/cx/v0.8/",
    "@vocab": "http://www.w3.org/ns/odrl.jsonld"
  },
  "@type": "Offer",
  "@id": "a343fcbf-99fc-4ce8-8e9b-148c97605aab",
  "permission": [
    {
      "action": "use",
      "constraint": {
        "and": [
          {
            "leftOperand": "cx:inForceDate",
            "operator": "gte",
            "rightOperand": {
              "@value": "contractAgreementStart",
              "@type": "cx:dateExpression"
            }
          },
          {
            "leftOperand": "cx:inForceDate",
            "operator": "lte",
            "rightOperand": {
              "@value": "contractAgreementStart + 100d",
              "@type": "cx:dateExpression"
            }
          }
        ]
      }
    }
  ]
}
```

## 2. Fixed Date

Fixed dates may also be specified as follows using `cx:inForceDate` operands:

```json
{
  "@context": {
    "cx": "https://w3id.org/cx/v0.8/",
    "@vocab": "http://www.w3.org/ns/odrl.jsonld"
  },
  "@type": "Offer",
  "@id": "a343fcbf-99fc-4ce8-8e9b-148c97605aab",
  "permission": [
    {
      "action": "use",
      "constraint": {
        "and": [
          {
            "leftOperand": "cx:inForceDate",
            "operator": "gte",
            "rightOperand": {
              "@value": "2023-01-01T00:00:01Z",
              "@type": "xsd:datetime"
            }
          },
          {
            "leftOperand": "cx:inForceDate",
            "operator": "lte",
            "rightOperand": {
              "@value": "2024-01-01T00:00:01Z",
              "@type": "xsd:datetime"
            }
          }
        ]
      }
    }
  ]
}
```

Although `xsd:datatime` supports specifying timezones, UTC should be used. It is an error to use an `xsd:datetime` without specifying the timezone.

## 3. No Period

If no period is specified the contract agreement is interpreted as having an indefinite in force period and will remain valid until its other constraints evaluate to false.

## 4. Not Before and Until

`Not Before` and `Until` semantics can be defined by specifying a single `cx:inForceDate` fixed date constraint and an appropriate operand. For example, the following policy
defines a contact is not in force before `January 1, 2023`:

 ```json
{
  "@context": {
    "cx": "https://w3id.org/cx/v0.8/",
    "@vocab": "http://www.w3.org/ns/odrl.jsonld"
  },
  "@type": "Offer",
  "@id": "a343fcbf-99fc-4ce8-8e9b-148c97605aab",
  "permission": [
    {
      "action": "use",
      "constraint": {
        "leftOperand": "cx:inForceDate",
        "operator": "gte",
        "rightOperand": {
          "@value": "2023-01-01T00:00:01Z",
          "@type": "xsd:datetime"
        }
      }
    }
  ]
}
```
