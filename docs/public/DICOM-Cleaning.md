# DICOM cleaning channel example (Mirth Connect)

The following guide outlines how to build a small set of Mirth Connect channels that pick up DICOM dump files from `/IB_DICOM`, normalize patient identifier tags, and forward the cleaned study to a PACS listening on AET `cleandicom` (`127.0.1.2:104`).

## Prerequisites

* Deploy the **dcm4chee/dcm4che** libraries the channel will call at runtime. The simplest approach is to drop the required JARs (for example `dcm4che-core`, `dcm4che-io`, `dcm4che-net`) into Mirth Connect's `custom-lib` directory and restart the service so they are available to the default library resource.
* Confirm the PACS accepts associations from your BridgeLink/Mirth host and that firewall rules allow outbound traffic to `127.0.1.2:104`.

## Channel layout

Create two channels (or one channel with a single destination, if preferred):

1. **DICOM File Inbound** – A File Reader source that pulls `.dcm` files from `/IB_DICOM`, updates patient identifiers inside a JavaScript transformer, and drops the transformed bytes on the channel map.
2. **Clean DICOM to PACS** – A DICOM Sender destination that reads the transformed payload from the channel map and transmits it to the PACS at AET `cleandicom`.

### Source: DICOM File Inbound

| Setting | Value |
| --- | --- |
| Connector type | File Reader |
| Polling | Interval (e.g., 5 seconds) |
| File Filter | `*.dcm` |
| Source Directory | `/IB_DICOM` |
| Move-To / Archive | Optional: set to `/IB_DICOM/archive` after successful send |
| Data Type | Binary |
| After Read | `Delete` or `Move` (depending on retention policy) |

Add a JavaScript Transformer step to convert and patch the dataset. The script below runs in the Rhino engine bundled with Mirth Connect and uses dcm4che classes to read/update/write the DICOM:

```javascript
// Access file metadata from the connector map
var sourceDir = connectorMessage.getConnectorMap().get('originalDirectory');
var filename = connectorMessage.getConnectorMap().get('originalFilename');
var filePath = sourceDir + java.io.File.separator + filename;

// dcm4che imports
var File = Packages.java.io.File;
var FileInputStream = Packages.java.io.FileInputStream;
var ByteArrayOutputStream = Packages.java.io.ByteArrayOutputStream;
var DicomInputStream = Packages.org.dcm4che3.io.DicomInputStream;
var DicomOutputStream = Packages.org.dcm4che3.io.DicomOutputStream;
var Tag = Packages.org.dcm4che3.data.Tag;
var VR = Packages.org.dcm4che3.data.VR;

var input = new DicomInputStream(new FileInputStream(new File(filePath)));
var dataset = input.readDataset(-1, -1);
input.close();

// Example: normalize patient identifiers. Replace values as appropriate for your site.
dataset.setString(Tag.PatientID, VR.LO, 'CLEAN-' + dataset.getString(Tag.PatientID));
dataset.setString(Tag.IssuerOfPatientID, VR.LO, 'BridgeLink');
dataset.setString(Tag.PatientName, VR.PN, dataset.getString(Tag.PatientName));
dataset.setString(Tag.OtherPatientIDs, VR.LO, dataset.getString(Tag.OtherPatientIDs));
dataset.setString(Tag.PatientBirthDate, VR.DA, dataset.getString(Tag.PatientBirthDate));

// Write the updated dataset back to bytes
var baos = new ByteArrayOutputStream();
var dos = new DicomOutputStream(baos, null);
dos.writeDataset(dataset.createFileMetaInformation(null), dataset);
dos.finish();
dos.close();

// Store the transformed bytes for downstream destinations
channelMap.put('cleanedDicomBytes', baos.toByteArray());
```

Notes:

* The `originalDirectory` and `originalFilename` variables are populated by the File Reader connector. If you prefer, you can set the source to **Leave File** and stream from `message` instead.
* Extend the tag list as needed; `Tag` and `VR` constants cover all identifier fields, such as `Tag.OtherPatientIDsSequence`, `Tag.PatientIdentityRemoved`, or issuer sequences.
* If the PACS requires a specific SOP Class UID or AE Title normalization, set those attributes before writing the dataset.

### Destination: Clean DICOM to PACS

| Setting | Value |
| --- | --- |
| Connector type | DICOM Sender |
| Remote AET | `cleandicom` |
| Host | `127.0.1.2` |
| Port | `104` |
| Local AET | (pick an AE title registered on the PACS) |
| Message | Use JavaScript: `channelMap.get('cleanedDicomBytes')` |
| Data Type | Binary |

If you created a separate channel for sending, add a Channel Writer destination to **DICOM File Inbound** that forwards to the **Clean DICOM to PACS** channel. Otherwise, simply add the DICOM Sender as a destination on the inbound channel and reference the `cleanedDicomBytes` map variable.

## Example channel export (XML)

You can paste the following XML into **Channel Tasks → Import Channel** to create a starting point with the steps above. Update AE Titles, host, and any site-specific tag logic before enabling the channels.

```xml
<channels version="3.12.0">
  <channel>
    <id>91f6d6a8-cc81-4fb7-9e9c-62e7848e5d14</id>
    <nextMetaDataId>0</nextMetaDataId>
    <enabled>false</enabled>
    <name>DICOM File Inbound</name>
    <description>Reads .dcm files, normalizes patient identifiers, and forwards cleaned bytes.</description>
    <sourceConnector>
      <mode>source</mode>
      <transportName>File Reader</transportName>
      <properties class="com.mirth.connect.connectors.file.FileReceiverProperties" version="3.12.0">
        <scheme>SFTP</scheme>
        <host></host>
        <port>22</port>
        <username></username>
        <password></password>
        <timeout>10000</timeout>
        <secure>true</secure>
        <passive>false</passive>
        <validateConnection>true</validateConnection>
        <afterReadAction>DELETE</afterReadAction>
        <afterReadMoveToDirectory></afterReadMoveToDirectory>
        <afterReadMoveToDirectoryRecursion>false</afterReadMoveToDirectoryRecursion>
        <errorResponseAction>MOVE</errorResponseAction>
        <errorResponseMoveToDirectory>/IB_DICOM/error</errorResponseMoveToDirectory>
        <errorResponseMoveToDirectoryRecursion>false</errorResponseMoveToDirectoryRecursion>
        <errorResponseMoveToFileName></errorResponseMoveToFileName>
        <errorResponseMoveToFileNameIncremental>true</errorResponseMoveToFileNameIncremental>
        <hostKeyChecking>automatic</hostKeyChecking>
        <knownHostsFilename></knownHostsFilename>
        <keyBasedAuthentication>false</keyBasedAuthentication>
        <keyFile></keyFile>
        <keyPassphrase></keyPassphrase>
        <fileFilter>*.dcm</fileFilter>
        <regex>false</regex>
        <directory>/IB_DICOM</directory>
        <ignoreDot>true</ignoreDot>
        <ignoreFileNameRegex></ignoreFileNameRegex>
        <checksum>false</checksum>
        <messageStorageMode>BINARY</messageStorageMode>
        <charset>UTF-8</charset>
      </properties>
    </sourceConnector>
    <destinationConnectors/>
    <preprocessingScript/>
    <postprocessingScript/>
    <deployScript/>
    <undeployScript/>
    <properties version="3.12.0"/>
    <filter/>
    <transformer version="3.12.0">
      <elements>
        <element>
          <sequenceNumber>0</sequenceNumber>
          <name>Normalize DICOM identifiers</name>
          <type>JavaScript</type>
          <data><![CDATA[// Access file metadata from the connector map
var sourceDir = connectorMessage.getConnectorMap().get('originalDirectory');
var filename = connectorMessage.getConnectorMap().get('originalFilename');
var filePath = sourceDir + java.io.File.separator + filename;

var File = Packages.java.io.File;
var FileInputStream = Packages.java.io.FileInputStream;
var ByteArrayOutputStream = Packages.java.io.ByteArrayOutputStream;
var DicomInputStream = Packages.org.dcm4che3.io.DicomInputStream;
var DicomOutputStream = Packages.org.dcm4che3.io.DicomOutputStream;
var Tag = Packages.org.dcm4che3.data.Tag;
var VR = Packages.org.dcm4che3.data.VR;

var input = new DicomInputStream(new FileInputStream(new File(filePath)));
var dataset = input.readDataset(-1, -1);
input.close();

dataset.setString(Tag.PatientID, VR.LO, 'CLEAN-' + dataset.getString(Tag.PatientID));
dataset.setString(Tag.IssuerOfPatientID, VR.LO, 'BridgeLink');

dataset.setString(Tag.PatientName, VR.PN, dataset.getString(Tag.PatientName));
dataset.setString(Tag.OtherPatientIDs, VR.LO, dataset.getString(Tag.OtherPatientIDs));
dataset.setString(Tag.PatientBirthDate, VR.DA, dataset.getString(Tag.PatientBirthDate));

dataset.setString(Tag.PatientIdentityRemoved, VR.CS, 'NO');

var baos = new ByteArrayOutputStream();
var dos = new DicomOutputStream(baos, null);
dos.writeDataset(dataset.createFileMetaInformation(null), dataset);
dos.finish();
dos.close();

channelMap.put('cleanedDicomBytes', baos.toByteArray());]]></data>
          <operator>equals</operator>
          <canRemove>false</canRemove>
        </element>
      </elements>
    </transformer>
    <exportData>false</exportData>
  </channel>
  <channel>
    <id>1c6f60b7-731d-4ad8-965e-cfbe7d8516e8</id>
    <nextMetaDataId>0</nextMetaDataId>
    <enabled>false</enabled>
    <name>Clean DICOM to PACS</name>
    <description>Sends cleaned DICOM objects to the PACS at AET cleandicom.</description>
    <sourceConnector>
      <mode>source</mode>
      <transportName>Channel Reader</transportName>
      <properties class="com.mirth.connect.connectors.vm.VmReceiverProperties" version="3.12.0"/>
    </sourceConnector>
    <destinationConnectors>
      <connector>
        <transportName>DICOM Sender</transportName>
        <metaDataId>0</metaDataId>
        <name>PACS cleandicom</name>
        <properties class="com.mirth.connect.connectors.dimse.DICOMDispatcherProperties" version="3.12.0">
          <hostname>127.0.1.2</hostname>
          <port>104</port>
          <applicationEntity>cleandicom</applicationEntity>
          <localApplicationEntity>CLEANSRC</localApplicationEntity>
          <messageTemplate>var bytes = channelMap.get('cleanedDicomBytes');
if (bytes == null) {
  throw 'No cleaned DICOM bytes found on channel map';
}
responseMap.put('payload', bytes);
return bytes;</messageTemplate>
        </properties>
      </connector>
    </destinationConnectors>
    <properties version="3.12.0"/>
  </channel>
</channels>
```

Import both channels, set the **DICOM File Inbound** destination to a Channel Writer pointing at **Clean DICOM to PACS**, and test with sample `.dcm` files placed in `/IB_DICOM`. Monitor the Mirth dashboard for connection status and PACS responses.
