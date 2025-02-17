# Creating Locations with Grid on Splinter

<!--
  Copyright (c) 2018-2020 Cargill Incorporated
  Licensed under Creative Commons Attribution 4.0 International License
  https://creativecommons.org/licenses/by/4.0/
-->

This procedure explains how to create and manage Grid locations
using Grid's command-line interface.

This procedure starts with steps to connect to a Splinter node and set
environment variables to simplify entering the `grid` commands in the procedure.
Next, it explains how to define a location with a location YAML file, create the
location with `grid location create`, and display location information with
`grid location list`.
Finally, this procedure shows how to update a location with `grid location
update`, then delete it with `grid location delete` when it is no longer
needed.

## Prerequisites

* Two (or more) nodes running Grid on Splinter.

* An approved Splinter circuit that has two (or more) member nodes.

* A fully qualified service ID for the scabbard service on this circuit, in the
  format `CircuitID::ServiceString`. This procedure uses the example ID
  `01234-ABCDE::gsAA`.

  Tip: Use `splinter circuit list` to display the circuit ID, then run `splinter
  circuit show CIRCUIT_ID` to see the four-character service string (also
  called the "partially qualified service ID").
  In the example Docker environment, you must run this command in the node's
  `splinterd` container.

* The Grid daemon's endpoint (URL and port) on one or both nodes.
  The example environment uses `https://localhost:8080`.

* The ID of an existing Pike organization that will own the locations. You
  must be defined as an agent with full location permissions for this
  organization. That is, one agent must be identified with your public key and
  must have permission to create, update, and delete locations.

  Tip: You can use `curl` to request organization and agent information from
  the Grid REST API, as in these examples:

  `$ curl https://localhost:8080/organization?service_id=01234-ABCDE::gsAA`

  `$ curl http://localhost:8080/agent?service_id=01234-ABCDE::gsAA`

* Existing public/private key files in `$HOME/.grid/keys/` (as generated by
  `grid keygen`). The example environment uses the key files
  `$HOME/.grid/keys/alpha-agent.priv` and `$HOME/.grid/keys/alpha-agent.pub` to
  indicate that you will be acting as your organization's agent to add locations
  from the alpha node.

* An existing schema for the new location. The schema name must be
  `gs1_location`, which tells Grid that the location schema will use the GS1
  namespace. For the commands to create a location schema, see
  [creating schemas]({% link
  docs/0.2/creating_schemas.md %}). A GS1-compliant schema is provided below:

  ```yaml
    - name: gs1_location
      description: GS1 location schema
      properties:
        - name: locationDescription
          data_type: String
          description: ""
          required: true
        - name: locationType
          data_type: Enum
          description: ""
          enum_options: [
            ShipTo, BillTo, ShipFrom, PaidBy, OrderFrom, Recall, OrgEntity, RemitTo
          ]
          required: true
        - name: addressLine1
          data_type: String
          description: ""
          required: true
        - name: addressLine2
          data_type: String
          description: ""
          required: false
        - name: city
          data_type: String
          description: ""
          required: true
        - name: stateOrRegion
          data_type: String
          description: ""
          required: true
        - name: postalCode
          data_type: String
          description: ""
          required: true
        - name: country
          data_type: String
          description: ""
          required: true
        - name: latLongValue
          data_type: lat_long
          description: ""
          required: true
        - name: contactName
          data_type: String
          description: ""
          required: true
        - name: contactEmail
          data_type: String
          description: ""
          required: true
        - name: contactPhone
          data_type: String
          description: ""
          required: true
        - name: createDate
          data_type: Number
          number_exponent: 0
          description: ""
          required: true
        - name: inactivationDate
          data_type: Number
          number_exponent: 0
          description: ""
          required: false
        - name: parentLocation
          data_type: String
          description: ""
          required: false
        - name: industrySector
          data_type: Enum
          description: ""
          enum_options: [General, CPG, Healthcare, Foodservice]
          required: false
        - name: role
          data_type: Enum
          description: ""
          enum_options: [
            Manufacturer, SolutionsProvider, Undefined, Distributor,
            Provider, Supplier, 3rdParty, Warhouse, IndependentOperator, Operator]
          required: false
        - name: informationProviderGLN
          data_type: Number
          number_exponent: 0
          description: ""
          required: false
  ```

## Procedure

**IMPORTANT**: The commands in this procedure show host names, IDs, Docker
container names, and other values from the example Grid-on-Splinter environment
that is defined by
[`grid/examples/splinter/docker-compose.yaml`](https://github.com/hyperledger/grid/blob/master/examples/splinter/docker-compose.yaml).
This file sets up the nodes `alpha-node-000` and `beta-node-000` and runs the
Grid and Splinter components in separate containers; these names appear in
example prompts and commands. (See [Running Hyperledger Grid on
Splinter]({% link docs/0.2/grid_on_splinter.md %}) for more information.)

If you are not using this example environment, replace these items with the
actual values for your environment when entering each command.

### Connect to a Grid Node

1. Connect to the first node's `gridd` container and start a bash session.

   For the example environment, use this command to connect to the `gridd-alpha`
   container and run Grid commands on `alpha-node-000`.

   ```
   $ docker exec -it gridd-alpha bash
   root@gridd-alpha:/#
   ```

### Set Up Your Environment

Set the following Grid environment variables to specify information for the
`grid` commands in this procedure.

{:start="2"}

2. Set `GRID_DAEMON_KEY` to the base name of your public/private key files
   (the example environment uses `alpha-agent`).  This environment variable
   replaces the `-k` option on the `grid` command line.

   ```
   root@gridd-alpha:/# export GRID_DAEMON_KEY="alpha-agent"
   ```

   **Note**: Although this variable has "daemon" in the name, it should
   reference the ***user key files*** in `$HOME/.grid/keys`, not the Grid
   daemon's key files in `/etc/grid/keys`.

1. Set `GRID_DAEMON_ENDPOINT` to the Grid daemon's endpoint (URL and port),
   such as `http://localhost:8080`. This environment variable replaces the
   `-url` option on the `grid` command line.

   ```
   root@gridd-alpha:/# export GRID_DAEMON_ENDPOINT="http://localhost:8080"
   ```

1. Set `GRID_SERVICE_ID` to the fully qualified service ID (such as
   `01234-ABCDE::gsAA`) for the scabbard service on the circuit. This
   environment variable replaces the `--service_id` option on the `grid`
   command line.

   ```
   root@gridd-alpha:/# export GRID_SERVICE_ID="01234-ABCDE::gsAA"
   ```

   Tip: See [Prerequisites](#prerequisites) for the `splinter` commands that
   display the circuit ID and four-character service ID string.

### Define and Create a Location

Each location requires an organization ID (the owner) and at least one agent
with permission to create, update, and delete the location. There must also
be an existing location schema for the location's properties.
See [Prerequisites](#prerequisites) for more information.

{:start="5"}

5. Create a location definition file, in YAML format, that specifies the
   following information:

   * location namespace (GS1 for this example) that corresponds with the
     location schema name (`gs1_location` for this example).
   * Location ID (such as a GLN)
   * Owner (organization ID)
   * Set of location attributes (each with a name and value that conforms to
     the data type)
     <br><br>

    ```yaml
    - namespace: "GS1"
      location_id: "0013600000011"
      owner: "myorg"
      properties:
        locationDescription: "test location"
        locationType: 0
        addressLine1: "251 Test St."
        city: "Minneapolis"
        stateOrRegion: "MN"
        postalCode: "55401"
        country: "United States"
        latLongValue: "449778,-932650"
        contactName: "John Tester"
        contactEmail: "tester@email.com"
        contactPhone: "6555555555"
        createDate: 1602706399
    ```

    Tip: Use a YAML linter to validate the new file is formatted correctly.

1. Use `docker cp` to copy the file into the `gridd-alpha` container.

   ```
   $ docker cp location.yaml gridd-alpha:/
   ```

1. Add the new location by using the `grid location create` command to specify the
   definition in the `location.yaml` file.

   ```
   root@gridd-alpha:/# grid location create --file location.yaml
   ```

   This command creates and submits a transaction to add the location data to the
   distributed ledger. If the transaction is successful, all other nodes in the
   circuit can view the location data.

   **Note**: This command does not display any output. Instead, check the log
   messages on the console (or the terminal window where you started the Grid
   Docker environment) for the success or failure of this operation.

### Display Location Information

{:start="8"}

8. List all existing locations to verify that the new location has been added.

    ```
    root@gridd-alpha:/# grid location list
    ID            NAMESPACE OWNER
    0013600000011 GS1       myorg
    ```

1. (Optional) You can connect to a different node and repeat the last two
   commands to verify that the location has been shared with all nodes on the
   circuit.

    a. Open a new terminal and connect to the other node's `gridd` container
       (such as `gridd-beta`).

      ```
      $ docker exec -it gridd-beta bash
      ```

    b. Set the `GRID_SERVICE_ID` environment variable to the full service ID
       for this node (such as `01234-ABCDE::xyBB`).

      ```
      root@gridd-beta:/# export GRID_SERVICE_ID="01234-ABCDE::xyBB"
      ```

    c. Display all locations. The output should be the same as on the first node.

    ```
    root@gridd-alpha:/# grid location list
    ID            NAMESPACE OWNER
    0013600000011 GS1       myorg
    ```

### Update a Location

To update a location, you must be an agent for the location owner (the
organization that is identified in the location definition).

1. Modify the definition in the location YAML file (such as `location.yaml`).

   You don't have to use the same file that was used to create the location,
   but the file must specify the same information (location ID and owner)
   and present the location properties in the same order.

1. Use the `update` subcommand for `grid location` to submit the changes
   to the distributed ledger.

   ```
   root@gridd-alpha:/# grid location update --file location.yaml
   ```

### Delete a Location

**IMPORTANT**: Deleting a location is a potentially hazardous operation that
must be done with care. Before you delete a location, make sure that no
member organizations on the circuit require the location data.

To delete a location, use the `delete` subcommand with the location ID and
namespace (default: `GS1`) (for example, ID `0013600000011`).

   ```
   root@gridd-alpha:/# grid location delete 0013600000011
   ```

Tip: Use `grid location list` to display the location ID and namespace.
