# Reproducing UTF8 issue #

This example is to show the UTF8 issue currently with the NAT Rest API call in combination with UTL_HTTP.

## Prerequisites ##

- Create a new Autonomous enviromment (19c)
- Create a new VCN in a (test) compartment
- Create a new SQL Developer Web session as ADMIN user

Gather the following information:

- Generic information
	- Tenancy OCID like `ocid1.tenancy.oc1..aaaaaaaa3fcdss2...kwpfbloh54cz7azsyjqgbl5qujnga`
- OCI User information
	- OCI user OCID like `ocid1.user.oc1..aaaaaaaaw4....kxr6tkvqmflgmfihfigrnkp6dmkj7q`
	- API KeyFingerprint like `5d:6d:....9:ec:69:df:f5`
- Test information
	- OCID of a (test) compartment like `ocid1.compartment.oc1..aaaaaaaaja5hm...tgzig5uvx7zljgwnenywjex4xmkcolq`
	- OCID of the new VCN like `ocid1.vcn.oc1.eu-frankfurt-1.amaaaaaaobogfhqavfwc...g2b5jvt7zk3m5e7pyyvg6a` 
	- Private key of used while creating the API Key

### Create CLOUD credentials under users schema ###

Inside the SQL Developer Web session, create a new credential by executing the following code (after replacing the required sections with your information):

````
<copy>begin
  DBMS_CLOUD.CREATE_CREDENTIAL ( credential_name => 'OCI_CRED',
                                  user_ocid       => 'ocid1.user.oc1..aaaaaaaaw4tkvowdatrjqlnijwmztykxr6tkddmflgmfihfigrnkp6dmkj7q',
                                  tenancy_ocid    => 'ocid1.tenancy.oc1..aaaaaaxx3fcdss2mnuqwijt4yrmkz4wkwpfbloh54cz7azsyjqgbl5quaaga',
                                  fingerprint     => '5d:6d:3f:3c:97:0d:d8:b3:f5:ef:5f:e9:ec:69:df:f5',
                                  private_key     => 'MIIEowIBAAK.....yqql3jC2e+DmvOYjru1kurK0iQB8MJr6z2j1Jx4NTKfZZTuY2tf4e8Ai8Yf3VrAk5mU1ehRQa8rirujoDuOECyzZqBCXaP.....Ltu9KPRvdq3X/'||
                                                     'z/MaDI4j7Yu0j5X0sa.....GIGHJewso3dZ8QyI/A7c7TDebXjK2BSJSxbh+AnCs92UpKu6GJpo9SSeM3n0sIg5iQ.....YWTCqE4KPr9aWSUD7kej9teoTRYEmUpPtD'||
                                                     'JvqywZv8MTVTiqc94oG.....RpSTXb92LIHiJVZuXNkqWgEb8H+dC+dY8Rfrryi3Uejtosjo9bA+BjBdIRm6Y56ojNEGdacbXOdvfwIDAQABAoIBADEtrag4C+ANmBAY'||
                                                     'jGojQwEm4jYofvpQcTIpVuvSGAoU5OTyzEm2hJ6K5i/T/Yn0nk+mGY7xJP6zMcSd0cnjHaSUkpB8zjDynidFdpiCgnJHs8eqj5PHeve2+jLnZr489i6yC1W5mp66++g2'||
                                                     ...
                                                     '1qXfVwKBgEXa/hP0cBiTJcxZeqUvyMH3LR7t6Zx9U4uReNP0JsENntbk6Kjz8pgI2aPfQ39V2eZzBCO8S6lJbdtGV6HNCLTNMNtFfjilq8EVu5RcoQXhZGOiasz8opVD'||
                                                     'KIiNUocZ1Yb8mEqV6bHJnZMpRibvoQ3m4eBV2+GMhnqqg0yGuWhy');
end;</copy>
````

The goal is to have your private pem key inside this statement. Please strip the values `-----BEGIN RSA PRIVATE KEY-----` and 
`-----END RSA PRIVATE KEY-----` from the private .pem file. 

> **Be aware:** if your source file does not start/ends with exactly the above lines, you do not have a PEM key but a RSA key.

Execute the PL/SQL command, you should get the following result:

    Statement processed.
    
    1.17 seconds

## Attempt to create a new NAT gateway ##

Using the gathered details, attempt to create a new NAT gateway by executing the following code (after replacing the applicable values by your values):

````
<copy>declare

  v_region          varchar2(200)  := '<your region'; --like 'eu-frankfurt-1';
  v_url             varchar2(200)  := 'https://iaas.{region}.oraclecloud.com';
  v_compartment_id  varchar2(100)  :=  '<Compartment ID of VCN>';
  v_display_name    varchar2(200)  := 'MyNatGateway';
  v_vcn_id          varchar2(200)  := '<VCN ID';
  v_credential      varchar2(200)  := 'OCI_CRED';

  v_method       varchar2(200);
  v_body         clob;
  v_response     DBMS_CLOUD_TYPES.resp;
  v_output       clob;


begin

  v_url    := replace(v_url,'{region}',v_region);
  v_method := '/20160918/natGateways';

  v_body := json_object ('compartmentId'  VALUE v_compartment_id,
                         'displayName'    VALUE v_display_name,
                         'vcnId'          VALUE v_vcn_id);
                            
  dbms_output.put_line(v_url||v_method);
  dbms_output.put_line(v_body);

  v_response := DBMS_CLOUD.send_request( credential_name => v_credential,
                                         uri             => v_url||v_method,
                                         method          => DBMS_CLOUD.METHOD_POST,
                                         body            => UTL_RAW.cast_to_raw(v_body)
                                       );

  v_output := DBMS_CLOUD.GET_RESPONSE_TEXT( resp => v_response );
  dbms_output.put_line(v_output);
                                         
end;
/</copy>

ORA-20000: ORA-44102: unknown or unsupported algorithm 
ORA-06512: at "C##CLOUD$SERVICE.DBMS_CLOUD", line 968 
ORA-06512: at "C##CLOUD$SERVICE.DBMS_CLOUD", line 2864 
ORA-06512: at line 28
````

Executing the same code for the second time gives the following result:

````
ORA-20400: Request failed with status HTTP 400 - https://iaas.eu-frankfurt-1.oraclecloud.com/20160918/natGateways 
Error response - { "code" : "LimitExceeded", "message" : "NAT gateway limit per VCN reached" }
ORA-06512: at "C##CLOUD$SERVICE.DBMS_CLOUD", line 964 
ORA-06512: at "C##CLOUD$SERVICE.DBMS_CLOUD", line 2864 
ORA-06512: at line 28
````

This means the code has been executed but the DBMS_CLOUD package (which depends on the UTL_HTTP package) has a problem with the return message after succesful execution.

## Retrieve headers while executing the request ##

- Open a Cloud Shell in the OCI Console in your region
- Delete the NAT gateway that has been created in previous step

Execute the following code (after replacing your applicable values in the code):

````
$ <copy>oci network nat-gateway create --compartment-id <your compartment ID> --vcn-id <your VCN ID> --display-name "MyNatGateway" --debug | grep header</copy>
````

Initial execution leads to the following headers:

````
header: Date: Fri, 01 May 2020 10:08:32 GMT
header: opc-request-id: 3E42E4F2394149018CC882F1C3760916/0BC9BFA475E4584FF6D1B2486D7BEE5A/E80B76B1AEB7BC635A7679C1198DB282
header: ETag: 367495352
header: Content-Encoding: UTF-8
header: Content-Type: application/json
header: X-Content-Type-Options: nosniff
header: Content-Length: 620
````

Second execution (which should error as it already exists) leads to the following headers:

````
header: Date: Fri, 01 May 2020 10:07:23 GMT
header: opc-request-id: C7D96BD352EC4E9799B7AF61AC7573F3/E145435B8AE7FA0EB8A141A8045EFF74/BA065C38C544642E38AEA17B44ADF4F5
header: Content-Type: application/json
header: X-Content-Type-Options: nosniff
header: Content-Length: 81
````

## Check on header options ##

https://tools.ietf.org/html/rfc7231#section-3.1.2.2

Content-Encoding is primarily used to allow a representation's data to be compressed without losing the identity of its underlying media type.

UTF8 is not a correct value
UTL_HTTP gives the correct error
The NAT gateway should be changed to not return the Content-EncodingL UTF8
