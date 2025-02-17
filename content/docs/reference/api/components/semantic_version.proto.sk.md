---
title: "semantic_version.proto"
weight: 10
---

<!-- Code generated by solo-kit. DO NOT EDIT. -->


**Package**: `solo.io.envoy.type.v3` 
**Types**:


- [SemanticVersion](#semanticversion)
  



**Source File**: [github.com/solo-io/gloo/projects/gloo/api/external/envoy/type/v3/semantic_version.proto](https://github.com/solo-io/gloo/blob/main/projects/gloo/api/external/envoy/type/v3/semantic_version.proto)





---
### SemanticVersion

 
Envoy uses SemVer (https://semver.org/). Major/minor versions indicate
expected behaviors and APIs, the patch version field is used only
for security fixes and can be generally ignored.

```yaml
"majorNumber": int
"minorNumber": int
"patch": int

```

| Field | Type | Description |
| ----- | ---- | ----------- | 
| `majorNumber` | `int` |  |
| `minorNumber` | `int` |  |
| `patch` | `int` |  |





<!-- Start of HubSpot Embed Code -->
<script type="text/javascript" id="hs-script-loader" async defer src="//js.hs-scripts.com/5130874.js"></script>
<!-- End of HubSpot Embed Code -->
