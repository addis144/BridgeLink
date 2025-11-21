# DICOM patient identifier transformer (Rhino JavaScript)

This example shows how to add a **JavaScript** step to a Mirth Connect **Source** transformer that updates DICOM patient-identifier tags (0010,0020 through 0010,1001) before the message is sent to destinations.

## Prerequisites
- Channel uses a **DICOM Listener** source and a destination that accepts a modified DICOM payload.
- The transformer step runs in the standard Rhino JavaScript context shipped with Mirth Connect.
- Input DICOM bytes arrive as Base64 text in `connectorMessage.getRawData()` (the default for binary connectors).

## Script
Insert a JavaScript step in the Source transformer with the script below. The helper function overwrites or removes common patient identifier tags based on values you place in the channel map (e.g., via global variables, database lookups, or static values in the transformer).

```javascript
// Rhino JavaScript: Source Transformer
var ByteArrayInputStream = Packages.java.io.ByteArrayInputStream;
var ByteArrayOutputStream = Packages.java.io.ByteArrayOutputStream;
var DicomInputStream = Packages.org.dcm4che2.io.DicomInputStream;
var DicomOutputStream = Packages.org.dcm4che2.io.DicomOutputStream;
var Tag = Packages.org.dcm4che2.data.Tag;
var VR = Packages.org.dcm4che2.data.VR;
var Base64 = Packages.org.apache.commons.codec.binary.Base64;

// Decode inbound DICOM from Base64 (typical for binary connectors)
var inboundBytes = Base64.decodeBase64(connectorMessage.getRawData());
var dis = new DicomInputStream(new ByteArrayInputStream(inboundBytes));
var dicom = dis.readDicomObject();
dis.close();

function putIdentifier(tag, vr, value) {
    var text = value == null ? "" : String(value).trim();
    if (text.length > 0) {
        dicom.putString(tag, vr, text);
    } else {
        dicom.remove(tag);
    }
}

// Populate values from channel map or static text
putIdentifier(Tag.PatientID, VR.LO, $('patientId')); // (0010,0020)
putIdentifier(Tag.IssuerOfPatientID, VR.LO, $('issuerOfPatientId')); // (0010,0021)
putIdentifier(Tag.PatientIDType, VR.CS, $('patientIdType')); // (0010,0022)
putIdentifier(Tag.OtherPatientIDs, VR.LO, $('otherPatientIds')); // (0010,1000)
putIdentifier(Tag.OtherPatientNames, VR.PN, $('otherPatientNames')); // (0010,1001)
// If you need to populate Other Patient IDs Sequence (0010,1002) or Issuer of Patient ID Qualifiers Sequence (0010,0024),
// build each item as a DicomObject and add it to a sequence manually.

// Serialize and hand back to the channel as Base64
var baos = new ByteArrayOutputStream();
var dos = new DicomOutputStream(baos);
dos.writeDicomFile(dicom);
dos.close();

var updatedBytes = baos.toByteArray();
connectorMessage.setRawData(new java.lang.String(Base64.encodeBase64(updatedBytes)));
```

### Notes
- The script removes a tag when the corresponding value is empty, preventing stale identifiers from leaking downstream.
- Populate the channel-map variables (`patientId`, `issuerOfPatientId`, etc.) earlier in the transformer to control final values.
- If your source provides the raw byte array directly (not Base64), skip the `Base64` encode/decode and use `connectorMessage.setProcessedRawData(updatedBytes);` instead.

