# fhir-lists
Sample of using FHIR Lists (_list, Functional/Current Lists, and $find)

Sometimes it is more convenient, more efficient, and more secure, to limit FHIR Searches per pre-defined "Lists" of Resources.

[Since v2025.1 we support several List-related features](https://docs.intersystems.com/irisforhealthlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXIHRN_new20251#HXIHRN_new20251_healthinterop) in our FHIR Server.

I will highlight these here, and provide some samples.

In general you can see [details about the List Resource](https://www.hl7.org/fhir/list.html) from the official FHIR documentation.

But here is a short description based on the above:

> The FHIR **List** Resource represents a flat (optionally ordered) collection of records used for clinical lists (e.g., allergies, meds, alerts, histories) and workflow management (e.g., tracking patients, teaching cases).  
> Lists may be **homogeneous** (single resource type) or **heterogeneous** (mixed types, e.g., a problem list spanning Conditions, AllergyIntolerances, Procedures).  
> Use List when you need a **curated/filtered set** that cannot be obtained via a simple query (e.g., “current” allergies vs all recorded allergies).  
> Querying a List yields a **human-curated point-in-time snapshot**, whereas querying the resource endpoint usually returns a broader, non-curated, “as of now” dataset.

In our latest versions (2025.1+) you can find new support for working with Lists:

*   **The \_list Search Parameter** 

See [the related FHIR docs](https://hl7.org/fhir/search.html#_list) for a full description. See [our related Docs](https://docs.intersystems.com/healthconnectlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRSTD_interactions#HXFHIRSTD_search_params) for details about the support available (specifically around Type-level vs. System-level searches).

Using this feature you can define for example a List of certain Resources, for example Encounters or Patients, etc. which you might want to Search according to them, without having to detail all of them as multiple Search Parameters.

For example I could define a List of Patients:

PUT /List cURL snippet

    curl --location --request PUT 'http://myserver/fhir/r4/List/OrgAPatients' \
    --header 'Content-Type: application/fhir+json' \
    --header 'Authorization: *******' \
    --data '{
      "resourceType": "List",
      "id":"OrgAPatients",
      "status":"current",
      "mode":"snapshot",
      "entry": [
          {
            "item": {
               "reference": "Patient/2"
            }
          },
          {
            "item": {
               "reference": "Patient/3"
            }
          },
          {
            "item": {
               "reference": "Patient/4"
            }
          }
        ]
    }'

And then Search for them like this:

GET /Patient?\_list cURL snippet

    curl --location 'http://myserver/fhir/r4/Patient?_list=OrgAPatients' \
    --header 'Content-Type: application/fhir+json' \
    --header 'Authorization: *****'

And get back a curated list of the Patients I wanted, instead of having to "mention" all of them in some multiple Search Parameters.

And of course there are many other use-cases.

*   **Functional Lists (including the Custom Operation $update-functional)**

A special kind of lists are Functional Lists (or "Current Resource" Lists).

See [the related FHIR docs](https://hl7.org/fhir/lifecycle.html#current) for a full description.

For your convenience here is a short description based on the above:

> Many clinical systems maintain **“current” patient lists** (for example, a current problem list and a current medication list), but FHIR cannot reliably infer “current-ness” by inspecting a single resource instance.  
> Using **Condition** as an example, the same resource type may be posted for multiple legitimate purposes (curated problem list entry, encounter complaint/diagnosis, diagnostic workflow context, or inbound referral data), and **Condition has no element** that cleanly distinguishes these usages.  
> Because distinguishing current vs. past would require **retrospective alteration** (creating integrity and digital-signature concerns), a normal search on **Condition** for a patient will return more than just the curated “current problems,” and limiting it to “current only” would hide other important Condition records.  
> Therefore, whether a Condition (or similar record) is on a patient’s “current list” could determined by whether it is **referenced from the appropriate List**.  
> Via the REST API, this is expressed via the **list search mechanism** using `_list` with standard functional list names (e.g., `GET [base]/AllergyIntolerance?patient=42&_list=$current-allergies`), and the server may support this without necessarily exposing a standalone List instance.  
> There are several "common" functional list names such as `$current-problems`, `$current-medications`, `$current-allergies`, and `$current-drug-allergies` (a subset of allergies).

In order to allow maintaining these Functional Lists, we defined a Custom Operation, called $update-functional, which allows for creating and updated these kinds of lists. See more details in [our Docs](https://docs.intersystems.com/healthconnectlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRSTD_operations#HXFHIRSTD_operation_functional_lists).

You can define a list of current allergies for example like this:

POST /List/$update-functional?for=...&name=\\$current-allergies cURL snippet

    curl --location 'http://fhirserver/fhir/r4/List/$update-functional?for=34&name=%5C%24current-allergies' \
    --header 'Content-Type: application/fhir+json' \
    --header 'Authorization: *****' \
    --data '{
      "resourceType": "List",
      "status":"current",
      "mode":"working",
      "subject":  { "reference":"Patient/34" },
      "entry": [
          {
            "item": {
               "reference": "AllergyIntolerance/159"
            }
          },
          {
            "item": {
               "reference": "AllergyIntolerance/160"
            }
          },
          {
            "item": {
               "reference": "AllergyIntolerance/161"
            }
          }
        ]
    }'
    
    

This will create/update the $current-allergies List, for a specific Patient (ID 34 in the example above).

Note I include the '`for=`' in the URL pointing to the Patient ID, and in the List I have '`subject`' referencing the Patient.

(Note also that for the dollar sign ($) I used a slash (\\) before it, i.e.: `\$`)

And now, I can ask for AllergyIntolerance Resource of this Patient, and instead of getting all of them, I can ask for just the "current" ones, as defined in the List above.

This would look like this:

GET /AllergyIntolerance?patient=...&\_list=\\$current-allergies cURL snippet

    curl --location 'http://fhirserver/fhir/r4/AllergyIntolerance?patient=34&_list=%5C%24current-allergies' \
    --header 'Authorization: *****'

And this would return a subset of this Patient's allergies, per the current allergies list.

Note we're using the same \_list Search Parameter mentioned previously, just this time instead of with a "Custom List", with a "Functional List".

Note you can control the names of the functional list names (and for each list, it's `subject` Search Parameter, and `subject` Resource Type; for example in the sample above the `subject` Search parameter was `patient` and the `subject` Resource Type was `Patient`), via the FHIR Endpoint configuration, specifically the "Interactions Strategy Settings", see [related Docs here](https://docs.intersystems.com/healthconnectlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRINS_server_install_configure#HXFHIRINS_server_install_configure_advanced_strategy). This looks like this:

![](/sites/default/files/inline/images/images/image(12420).png)

*   **$find Operation**

In addition, if you simply want to get the Functional List itself (for a particular subject, and of a particular type), you can use the $find operation.

See [the related FHIR docs](https://hl7.org/fhir/list-operation-find.html) for a full description. And see also [our related Docs](https://docs.intersystems.com/healthconnectlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRSTD_operations).

Here's an example, per the previous one:

/List/$find?patient=...&name=\\$current-allergies cURL snippet

    curl --location 'http://fhirserver/fhir/r4/List/$find?patient=34&name=%5C%24current-allergies' \
    --header 'Authorization: *****'

This will return the related $current-allergies list for this Patient, as defined above via the $update-functional function.

See the related Open Exchange app which includes a Postman Collection with the samples above (plus a little more) and instructions re running this against @Evgeny.Shvarov's FHIR server template docker container (indeed the sample above was created around this sample; with a slight change... see details in the app instructions).

A general note - all this functionality assumes you are using the, relatively newer, and current default, JsonAdvSQL [Storage Strategy](https://docs.intersystems.com/healthconnectlatest/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRINS_server_install_new#HXFHIRINS_server_install_new_strategy) for your Endpoint. (If relevant see [here regarding migrating from a legacy Strategy](https://docs.intersystems.com/healthconnect20253/csp/docbook/DocBook.UI.Page.cls?KEY=HXFHIRLegacy_SQL))
