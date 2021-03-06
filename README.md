# Apex Test Kit

![](https://img.shields.io/badge/version-3.1.0-brightgreen.svg) ![](https://img.shields.io/badge/build-passing-brightgreen.svg) ![](https://img.shields.io/badge/coverage-95%25-brightgreen.svg)

Apex Test Kit is a Salesforce library to help generate massive records for either Apex test classes, or sandboxes. It solves two pain points during record creation:

1. Establish arbitrary levels of many-to-one, one-to-many relationships.
2. Generate field values based on simple rules automatically.

------

### **3.1 Release Notes**

3.1 has some `breaking` changes, it shouldn't affect existing methods compilation, since the old APIs are still there. However some changes required to be made to prevent runtime errors.

1. **<a href="#lookup-field-keywords">Lookup Field Keywords</a>** such as `recordType()` and `profile()` can no longer be chained after  `.field()`.
2. **<a href="#entity-builder-factory">Entity Builder Factory</a>** is a proper name to reflect its responsibility, so ATK.FieldBuilder has been renamed to ATK.EntityBuilder, and has to be used with `.build(entityBuilder)`. ATK.FieldBuilder will be completely removed in the next version 3.2.

------

Imagine the complexity to generate the following sObjects and establish all the relationships in the diagram.

<p align="center">
<img src="docs/images/sales-objects.png#2020-5-31" width="400" alt="Sales Object Graph">
</p>


With ATK we can create them within just one Apex statement. Here, we are generating:

1. *200* accounts with names: `Name-0001, Name-0002, Name-0003...`
2. Each of the accounts has *2* contacts.
3. Each of the contacts has *1* opportunity via the OpportunityContactRole.
4. Also each of the accounts has *2* orders.
5. Also each of the orders belongs to *1* opportunity from the same account.

```java
ATK.SaveResult result = ATK.prepare(Account.SObjectType, 200)
    .field(Account.Name).index('Name-{0000}')
    .withChildren(Contact.SObjectType, Contact.AccountId, 400)
        .field(Contact.LastName).index('Name-{0000}')
        .field(Contact.Email).index('test.user+{0000}@email.com')
        .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
        .withChildren(OpportunityContactRole.SObjectType, OpportunityContactRole.ContactId, 400)
            .field(OpportunityContactRole.Role).repeat('Business User', 'Decision Maker')
            .withParents(Opportunity.SObjectType, OpportunityContactRole.OpportunityId, 400)
                .field(Opportunity.Name).index('Name-{0000}')
                .field(Opportunity.ForecastCategoryName).repeat('Pipeline')
                .field(Opportunity.Probability).repeat(0.9, 0.8)
                .field(Opportunity.StageName).repeat('Prospecting')
                .field(Opportunity.CloseDate).addDays(Date.newInstance(2020, 1, 1), 1)
                .field(Opportunity.TotalOpportunityQuantity).add(1000, 10)
                .withParents(Account.SObjectType, Opportunity.AccountId)
    .also(4)
    .withChildren(Order.SObjectType, Order.AccountId, 400)
        .field(Order.Name).index('Name-{0000}')
        .field(Order.EffectiveDate).addDays(Date.newInstance(2020, 1, 1), 1)
        .field(Order.Status).repeat('Draft')
        .withParents(Contact.SObjectType, Order.BillToContactId)
        .also()
        .withParents(Opportunity.SObjectType, Order.OpportunityId)
    .save(true);
```

`withChildren()` and `withParents()` without a third size parameter indicate they will back reference the sObjects created previously in the statement. If the third size param is supplied, new sObjects will be created.

### Performance

To generate the above 2200 records and saving them into Salesforce, it will take less than 3000 CPU time. That's already 1/3 of the Maximum CPU time. However, if we use `.save(false)` without saving them, and just create them in the memory, it will take less than 700 CPU time, 4x faster.

### Installation

| Environment           | Install Link                                                 | Version   |
| --------------------- | ------------------------------------------------------------ | --------- |
| Production, Developer | <a target="_blank" href="https://login.salesforce.com/packaging/installPackage.apexp?p0=04t2v000007X3Q7AAK"><img src="docs/images/deploy-button.png"></a> | ver 3.1.0 |
| Sandbox               | <a target="_blank" href="https://test.salesforce.com/packaging/installPackage.apexp?p0=04t2v000007X3Q7AAK"><img src="docs/images/deploy-button.png"></a> | ver 3.1.0 |

### Demos

There are four demos under the `scripts/apex` folder, they can be successfully run in a clean Salesforce CRM organization. If not, please try to fix them with FLS, validation rules or duplicate rules etc.

| Subject  | File Path                         | Description                                                  |
| -------- | --------------------------------- | ------------------------------------------------------------ |
| Campaign | `scripts/apex/demo-campaign.apex` | How to genereate campaigns with hierarchy relationships. `ATK.FieldBuilder` is implemented to reuse the field population logic. |
| Cases    | `scripts/apex/demo-cases.apex`    | How to generate Accounts, Contacts and Cases.                |
| Products | `scripts/apex/demo-products.apex` | How to generate Products for standard Price Book.            |
| Sales    | `scripts/apex/demo-sales.apex`    | You've already seen it in the above paragraph.               |
| Users    | `scripts/apex/demo-users.apex`    | How to generate community users in one goal.                 |

## Relationship

### One to Many

```java
ATK.prepare(Account.SObjectType, 10)
    .field(Account.Name).index('Name-{0000}')
    .withChildren(Contact.SObjectType, Contact.AccountId, 20)
        .field(Contact.LastName).index('Name-{0000}')
        .field(Contact.Email).index('test.user+{0000}@email.com')
        .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
    .save();
```

Children will be evenly distributed among parents, and here is how the relationship going to be mapped:

| Account Name | Contact Name |
| ------------ | ------------ |
| Name-0001    | Name-0001    |
| Name-0001    | Name-0002    |
| Name-0002    | Name-0003    |
| Name-0002    | Name-0004    |
| ...          | ...          |

### Many to One

```java
ATK.prepare(Contact.SObjectType, 20)
    .field(Contact.LastName).index('Name-{0000}')
    .field(Contact.Email).index('test.user+{0000}@email.com')
    .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
    .withParents(Account.SObjectType, Contact.AccountId, 10)
        .field(Account.Name).index('Name-{0000}')
    .save();
```

### Many to Many

```java
ATK.prepare(Contact.SObjectType, 40)
    .field(Contact.LastName).index('Name-{0000}')
    .field(Contact.Email).index('test.user+{0000}@email.com')
    .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
    .withChildren(OpportunityContactRole.SObjectType, OpportunityContactRole.ContactId, 40)
        .field(OpportunityContactRole.Role).repeat('Business User', 'Decision Maker')
        .withParents(Opportunity.SObjectType, OpportunityContactRole.OpportunityId, 40)
            .field(Opportunity.Name).index('Name-{0000}')
            .field(Opportunity.CloseDate).addDays(Date.newInstance(2020, 1, 1), 1)
            .field(Opportunity.StageName).repeat('Prospecting')
    .save();
```

## Keywords And APIs

There are only two keyword categories Entity keyword and Field keyword. They are used to solve the two pain points we addressed at the beginning:

1. **Entity Keyword**: Establish arbitrary levels of many-to-one, one-to-many relationships.
2. **Field Keyword**: Generate field values based on simple rules automatically.

### Entity Keywords

Here is a dummy example to demo the use of Entity keywords. Each of them will start a new sObject context. And it is advised to use the following indentation for clarity.

```java
ATK.prepare(A__c.SObjectType, 10)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, 10)
        .withParents(C__c.SObjectType, B__c.C_ID__c, 10)
            .withChildren(D__c.SObjectType, D__c.C_ID__c, 10)
            .also() // Go back 1 depth to C__c
            .withChildren(E__c.SObjectType, E__c.C_ID__c, 10)
        .also(2)    // Go back 2 depth to B__c
        .withChildren(F__c.SObjectType, F__c.B_ID__c, 10)
    .save();
```

#### Entity Create Keywords

All the following APIs with a `Integer size` param at the last, indicate how many of the associated sObject type will be created on the fly.

```java
ATK.prepare(A__c.SObjectType, 10)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, 10)
        .withParents(C__c.SObjectType, B__c.C_ID__c, 10)
    .save();
```

| Keyword API                                                  | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| prepare(SObjectType *objectType*, Integer *size*)            | Always start chain with `prepare()` keyword. It is the root sObject to start relationship with. |
| withChildren(SObjectType *objectType*, SObjectField *referenceField*, Integer *size*) | Establish one to many relationship between the previous working on sObject and the current sObject. |
| withParents(SObjectType *objectType*, SObjectField *referenceField*, Integer *size*) | Establish many to one relationship between the previous working on sObject and the current sObject. |

#### Entity Update Keywords

All the following APIs with a `List<SObject> objects` param at the last, indicate the sObjects are created elsewhere, and ATK just upsert them.

```java
ATK.prepare(A__c.SObjectType, [SELECT Id FROM A__c]) // Select existing sObjects
    .field(A__c.Name).index('Name-{0000}')           // Update existing sObjects
    .field(A__c.Price).repeat(100)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, new List<SObject> {
        new B__c(Name = 'Name-A'),                   // Manually assign field values
        new B__c(Name = 'Name-B'),
        new B__c(Name = 'Name-C')})
        .field(B__c.Counter__c).add(1, 1)            // Automatically assign field values
        .field(B__c.Weekday__c).repeat('Mon', 'Tue') // Automatically assign field values
        .withParents(C__c.SObjectType, B__c.C_ID__c, new List<SObject> {
            new C__c(Name = 'Name-A'),
            new C__c(Name = 'Name-B'),
            new C__c(Name = 'Name-C')})
    .save();
```

| Keyword API                                                  | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| prepare(SObjectType *objectType*, List\<SObject\> *objects*) | Always start chain with `prepare()` keyword. It is the root sObject to start relationship with. |
| withChildren(SObjectType *objectType*, SObjectField *referenceField*, List\<SObject\> *objects*) | Establish one to many relationship between the previous working on sObject and the current sObject. |
| withParents(SObjectType *objectType*, SObjectField *referenceField*, List\<SObject\> *objects*) | Establish many to one relationship between the previous working on sObject and the current sObject. |

#### Entity Reference Keywords

All the following APIs without a third param at the last, indicate a back reference to the previously created sObjects within the statement. Thus, no new records will be created with the following statements.

| Keyword API                                                  | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| withChildren(SObjectType *objectType*, SObjectField *referenceField*) | Establish one to many relationship between the previous working on sObject and the current sObject. |
| withParents(SObjectType *objectType*, SObjectField *referenceField*) | Establish many to one relationship between the previous working on sObject and the current sObject. |

**Note**: Once these APIs are used, please make sure there are sObjects with the same type created previously, and only created once.

### Field Keywords

Here is a dummy example to demo the use of field keywords.

```java
ATK.prepare(A__c.SObjectType, 10)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, 10)
        .field(B__C.Name__c).index('Name-{0000}')
        .field(B__C.PhoneNumber__c).index('+86 186 7777 {0000}')
        .field(B__C.Price__c).repeat(12.34)
        .field(B__C.CampanyName__c).repeat('Google', 'Apple', 'Microsoft')
        .field(B__C.Counter__c).add(1, 1)
        .field(B__C.StartDate__c).addDays(Date.newInstance(2020, 1, 1), 1)
    .save();
```
#### Basic Field Keywords
| Keyword API                                               | Description                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| index(String *format*)                                    | Formated string with `{0000}`, can recogonize left padding. For example: 0001, 0002, 0003 etc. |
| repeat(Object *value*)                                    | Repeat with a fixed value.                                   |
| repeat(Object *value1*, Object *value2*)                  | Repeat with the provided values alternatively.               |
| repeat(Object *value1*, Object *value2*, Object *value3*) | Repeat with the provided values alternatively.               |
| repeat(List\<Object\> *values*)                           | Repeat with the provided values alternatively.               |

#### Lookup Field Keywords

These are field keywords in nature, but don't need to be chained after `.field(Schema.SObjectField)`. Because ATK automatically helps resolving the correct fields for them.

```java
ATK.prepare(User.SObjectType, 10)
    .profile('Chatter Free User')      // must be applied to User SObject
    .permissionSet('Survey Creator');  // must be applied to User SObject

ATK.prepare(Account.SObjectType, 10)
    .recordType('Business Account');
```

| Keyword API                                                  | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| recordType(String *name*)                                    | Assign record type ID by developer name or label             |
| profile(String *name*)                                       | Assign profile ID by profile name.                           |
| permissionSet(String *name*)                                 | Assign the permission set to users by its developer name or label. |
| permissionSet(String name1, String *name2*)                  | Assign all the permission sets to users by developer name or label. |
| permissionSet(String *name1*, String *name2*, String *name3*) | Assign all the permission sets to users by developer name or label. |
| permissionSet(List\<String\> *names*)                        | Assign all the permission sets to users by developer name or label. |

#### Arithmetic Field Keywords

These keywords will increase/decrease the init values by the provided step values. They must be applied to the correct field data types that support them.

| Keyword API                                | Description                                    |
| ------------------------------------------ | ---------------------------------------------- |
| add(Decimal *init*, Decimal *step*)        | Must be applied to a number type field.        |
| substract(Decimal *init*, Decimal *step*)  | Must be applied to a number type field.        |
| divide(Decimal *init*, Decimal *factor*)   | Must be applied to a number type field.        |
| multiply(Decimal *init*, Decimal *factor*) | Must be applied to a number type field.        |
| addYears(Object *init*, Integer *step*)    | Must be applied to a Datetime/Date type field. |
| addMonths(Object *init*, Integer *step*)   | Must be applied to a Datetime/Date type field. |
| addDays(Object *init*, Integer *step*)     | Must be applied to a Datetime/Date type field. |
| addHours(Object *init*, Integer *step*)    | Must be applied to a Datetime/Time type field. |
| addMinutes(Object *init*, Integer *step*)  | Must be applied to a Datetime/Time type field. |
| addSeconds(Object *init*, Integer *step*)  | Must be applied to a Datetime/Time type field. |

## Entity Builder Factory

It is recommended to keep how the sObject relationship is established in the test class. Because they are less subject to change, and clearer to the developers. In order to increase the reusability, let's introduce the Entity Builder Factory:

```java
@IsTest
public with sharing class CampaignServiceTest {
    @TestSetup
    static void setup() {
        ATK.SaveResult result = ATK.prepare(Campaign.SObjectType, 4)
            .build(EntityBuilderFactory.campaignBuilder)     // Reference to Entity Builder
            .withChildren(CampaignMember.SObjectType, CampaignMember.CampaignId, 8)
                .withParents(Lead.SObjectType, CampaignMember.LeadId, 8)
                    .build(EntityBuilderFactory.leadBuilder) // Reference to Entity Builder
            .save();
    }
}
```

A field builder factory example:

```java
@IsTest
public with sharing class EntityBuilderFactory {
    public static CampaignEntityBuilder campaignBuilder = new CampaignEntityBuilder();
    public static LeadEntityBuilder leadBuilder = new LeadEntityBuilder();

    // Inner class implements ATK.FieldBuilder
    public class CampaignEntityBuilder implements ATK.EntityBuilder {
        public void build(ATK.Entity campaignEntity, Integer size) {
            campaignEntity
                .field(Campaign.Type).repeat('Partners')
                .field(Campaign.Name).index('Name-{0000}')
                .field(Campaign.StartDate).repeat(Date.newInstance(2020, 1, 1))
                .field(Campaign.EndDate).repeat(Date.newInstance(2020, 1, 1).addMonths(1))
        }
    }

    // Inner class implements ATK.FieldBuilder
    public class LeadEntityBuilder implements ATK.EntityBuilder {
        public void build(ATK.Entity leadEntity, Integer size) {
            leadEntity
                .field(Lead.Company).index('Name-{0000}')
                .field(Lead.LastName).index('Name-{0000}')
                .field(Lead.Email).index('test.user+{0000}@email.com')
                .field(Lead.MobilePhone).index('+86 186 7777 {0000}')
        }
    }
}
```

## License

Apache 2.0
