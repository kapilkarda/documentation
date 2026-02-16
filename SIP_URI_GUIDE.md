# SIP URI Configuration Guide

## Overview
The SIP URI has been centralized using Mintlify's snippet feature for easy maintenance across all documentation files.

## Current Configuration
- **SIP URI**: `sip:31.220.77.222:5060`
- **Snippet File**: `docs/snippets/sip-uri.mdx`

## Files Updated
The following documentation files now reference the centralized SIP URI snippet:
1. `docs/general/sip/vonage.mdx`
2. `docs/general/sip/telnyx.mdx`
3. `docs/general/sip/twilio.mdx`

## How to Update the SIP URI in the Future

### Step 1: Edit the Snippet File
Simply update the content in `docs/snippets/sip-uri.mdx`:

```mdx
sip:YOUR_NEW_SIP_URI_HERE
```

### Step 2: That's It!
All three documentation files will automatically use the new SIP URI since they reference the snippet.

## How It Works
Each documentation file uses the Mintlify `<Snippet>` component:

```mdx
<code><Snippet file="sip-uri.mdx" /></code>
```

This component automatically includes the content from `docs/snippets/sip-uri.mdx`, ensuring consistency across all pages.

## Benefits
✅ **Single Source of Truth**: Update once, changes everywhere  
✅ **No Risk of Inconsistency**: All pages always show the same SIP URI  
✅ **Easy Maintenance**: No need to search and replace across multiple files  
✅ **Version Control Friendly**: Clear history of SIP URI changes
