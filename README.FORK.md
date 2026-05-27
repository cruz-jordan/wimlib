# Additional APIs in this fork

## XML Utilities

This fork exposes one extra public helper for accessing a WIM's XML metadata
in a managed-friendly way. Plus a corresponding free function:

### [wimlib_get_wim_xml](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L3387)
```c
/**
 * @ingroup G_wim_information
 * 
 * Load a WIM file's XML document into memory. Similar to wimlib_get_xml_data(),
 * but the XML is returned as a "wimlib_tchar" string.
 *
 * @param wim
 *  Pointer to the ::WIMStruct to query. This need not represent a
 *	standalone WIM (e.g. it could represent part of a split WIM).
 * @param xml_ret
 *  On success, will point to a newly allocated, null-terminated
 *  wimlib_tchar string containing the XML document. The caller must free it
 *  with wimlib_free_tstr().
 *
 * @return 0 on success; a ::wimlib_error_code value on failure. This may
 *  also return any error code which can be returned by wimlib_get_xml_data().
 */
WIMLIBAPI int
wimlib_get_wim_xml(const WIMStruct *wim, wimlib_tchar **xml_ret);
```

[xml.c](https://github.com/cruz-jordan/wimlib/blob/master/src/xml.c#L1110)
```c
/*
* Return the WIM's XML document as a wimlib_tchar* string.
* Caller must free *xml_ret using wimlib_free_tstr().
*/
WIMLIBAPI int
wimlib_get_wim_xml(const WIMStruct *wim, wimlib_tchar **xml_ret)
{
	void *raw_doc = NULL;
	size_t raw_doc_size = 0;
	int ret;

	if (!xml_ret)
		return WIMLIB_ERR_INVALID_PARAM;

	*xml_ret = NULL;

	/* Existing API, read raw UTF-16LE XML into raw_doc + size in utf16lechar count */
	ret = wimlib_get_xml_data((WIMStruct *)wim, &raw_doc, &raw_doc_size);
	if (ret)
		return ret;

	ret = utf16le_to_tstr((const utf16lechar *)raw_doc, 
					raw_doc_size, 
					xml_ret,
					NULL);

	FREE(raw_doc);

	if (ret) {
		/* utf16le_to_tstr guarantees *xml_ret is either NULL or valid */
		return ret;
	}

	return 0;
}
```

### [wimlib_free_tstr](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L3250)
```c
/**
 * @ingroup G_general
 * 
 * Free a "wimlib_tchar" string previously allocated by either wimlib_get_wim_xml()
 * or wimlib_load_text_file().
 * 
 * @param tstr
 *  Pointer to the wimlib_tchar string to release. If @c NULL, no action is taken.
 */
WIMLIBAPI void
wimlib_free_tstr(wimlib_tchar *tstr);
```

[textfile.c](https://github.com/cruz-jordan/wimlib/blob/master/src/textfile.c#L406)
```c
/* Free a wimlib_tchar string allocated by wimlib_get_wim_xml() or wimlib_load_text_file() */
WIMLIBAPI void
wimlib_free_tstr(wimlib_tchar *tstr)
{
	if (tstr)
		FREE(tstr);
}
```

### Rationale

The upstream API already provides:
```c
WIMLIBAPI int
wimlib_get_xml_data(WIMStruct *wim, void **buf_ret, size_t *bufsize_ret);
```
Which returns the raw UTF-16LE XML blob and leaves it to callers to convert
it into their preferred string encoding, and ensure the buffer is freed
correctly.

For .NET/FFI scenarios, this leads to extra interop glue and manual memory
management for each call. These additional helpers wrap this pattern so
that:

- The XML document is converted into a single `wimlib_tchar*` string
using wimlib's own `utf16le_to_tstr()` helper.
- Ownership and lifetime are well-defined, the caller always releases the
string via `wimlib_free_tstr()`.

This makes it trivial forr managed bindings (e.g., C#) to:

- Call `wimlib_get_wim_xml()` once.
- Marshal the returned `wimlib_tchar*` to a managed string.
- Parse it using standard XML tooling.
- Call `wimlib_free_tstr()` to release the native allocation.

## WIM Info Changes

The following field has been added to the WIM information struct for easier
determination of whether a WIM contains solid resources or not:

### [wimlib_wim_info](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L1408)
```c
/** 1 iff this WIM contains contains solid resources.  */
	uint32_t packed_resources : 1;
	uint32_t reserved_flags : 21;
```

[wim.c](https://github.com/cruz-jordan/wimlib/blob/master/src/wim.c#L491)
```c
	info->packed_resources = wim_has_solid_resources(wim);
```

### Rationale

A WIM which contains solid resources cannot be split via the `wimlib_split()` function
and this provides a simple way for checking beforehand through the existing `wim_has_solid_resources()`
function in the upstream source.

## Compression Level Utilities

The below functions have been added for controlling the output compression level of individual
WIMStruct's:

### [wimlib_set_output_compression_level](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L4302)
```c
/**
 * @ingroup G_writing_and_overwriting_wims
 *
 * Set a ::WIMStruct's output compression level. This is the compression
 * level that will be used for writing non-solid resources in subsequent
 * calls to wimlib_write() or wimlib_overwrite() for the WIM's output
 * compression type.
 * 
 * The initial state, before this function is called, is that all compression
 * types have a default compression level of 50.
 * 
 * @param wim
 *	The ::WIMStruct for which to set the output compression level.
 * 
 * @param compression_level
 *	The compression level to set. If 0, the "default" level
 *	of 50 is restored.  Otherwise, a higher value indicates higher
 *	compression, whereas a lower value indicates lower compression.
 *  The values are scaled so that 10 is low compression, 50 is medium
 *  compression, and 100 is high compression. This is not a percentage;
 *  values above 100 are also valid.
 * 
 * @return 0 on success; a ::wimlib_error_code value on failure.
 * 
 * @retval ::WIMLIB_ERR_INVALID_PARAM
 *  @p compression_level was an unsupported level.
 */
WIMLIBAPI int
wimlib_set_output_compression_level(WIMStruct *wim,
				   unsigned int compression_level);
```

### [wimlib_set_output_pack_compression_level](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L4332)
```c
/**
 * @ingroup G_writing_and_overwriting_wims
 *
 * Similar to wimlib_set_output_compression_level(), but sets the output 
 * compression level for writing solid resources.
 */
WIMLIBAPI int
wimlib_set_output_pack_compression_level(WIMStruct *wim,
						unsigned int compression_level);	
```

[wim.h](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib/wim.h#L142)
```c
	/* Overridden compression level for wimlib_overwrite() or wimlib_write().
	 * 0 means use the existing default behavior.  */
	u32 out_compression_level;

	/* Overridden compression level for writing solid resources.
	 * 0 means use the existing default behavior.  */
	u32 out_solid_compression_level;
```

[wim.c](https://github.com/cruz-jordan/wimlib/blob/master/src/wim.c#L567)
```c
/* API function documented in wimlib.h  */
WIMLIBAPI int
wimlib_set_output_compression_level(WIMStruct *wim,
					unsigned int compression_level)
{
	if (compression_level & WIMLIB_COMPRESSOR_FLAG_DESTRUCTIVE)
		return WIMLIB_ERR_INVALID_PARAM;

	if (compression_level > 0x00FFFFFF)
		return WIMLIB_ERR_INVALID_PARAM;

	wim->out_compression_level = compression_level;
	return 0;
}

/* API function documented in wimlib.h  */
WIMLIBAPI int
wimlib_set_output_pack_compression_level(WIMStruct *wim,
						 unsigned int compression_level)
{
	if (compression_level & WIMLIB_COMPRESSOR_FLAG_DESTRUCTIVE)
		return WIMLIB_ERR_INVALID_PARAM;

	if (compression_level > 0x00FFFFFF)
		return WIMLIB_ERR_INVALID_PARAM;

	wim->out_solid_compression_level = compression_level;
	return 0;
}
```

### Rationale

Previously, the compression level utilized when writing or overwriting a WIMStruct 
was only configurable via the `wimlib_set_default_compression_level()` function, which
would effect the compression level for the specified compression type globally. Consequently,
this would result in potential misconfigurations in the case of separate WIMStructs being
written to disk concurrently.

## Directory Entry Utilities

The following utility function for testing whether a directory entry exists for a given 
path has been added:

### [wimlib_dir_entry_exists](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L2835)
```c
/**
 * @ingroup G_wim_information
 *
 * Determine whether a directory entry exists at the specified @p path
 * for a WIM image.
 *
 * @param wim
 * 	The ::WIMStruct containing the image for which to query. The ::WIMStruct
 * 	must contain image metadata, so in the case of split WIMs, this should be
 * 	first part.
 * 
 * @param image
 * 	The 1-based index of the image for which to query.
 * 
 * @param path
 * 	Path in the image for which to test.
 * 
 * @return 0 if a directory entry exists at the specified path; otherwise -1 if
 * no directory entry exists at the path in the image. The following additional
 * ::wimlib_error_code values may also be returned:
 * 
 * @retval ::WIMLIB_ERR_INVALID_IMAGE
 * 	@p image does not exist in @p wim.
 * 
 * @retval ::WIMLIB_ERR_NOMEM
 * 	Insufficient memory to perform the query.
 */
WIMLIBAPI int
wimlib_dir_entry_exists(WIMStruct *wim, int image, const wimlib_tchar *path);
```

[dentry.c](https://github.com/cruz-jordan/wimlib/blob/master/src/dentry.c#L1910)
```c
static int
image_do_dentry_exists(WIMStruct *wim)
{
	const tchar *path = wim->private;

	return get_dentry(wim, path, WIMLIB_CASE_PLATFORM_DEFAULT) ? 0 : -1;
}

/* Determine whether a directory entry exists at a specified path for an image.  */
WIMLIBAPI int
wimlib_dir_entry_exists(WIMStruct *wim, int image, const tchar *_path)
{
	tchar *path;
	int ret;

	path = canonicalize_wim_path(_path);
	if (path == NULL)
		return WIMLIB_ERR_NOMEM;

	wim->private = path;
	ret = for_image(wim, image, image_do_dentry_exists);
	FREE(path);
	return ret;
}
```

### Rationale

This new function adds a simple way for testing path existence within a WIM image
using existing utilities from the upstream source without the need for iteration
via `wimlib_iterate_dir_tree()` which may potentially return `WIMLIB_ERR_PATH_DOES_NOT_EXIST`,
providing callers a way to potentially avoid such errors.

# Fork behaviour and compatability

No existing functions or structures have been removed, or modified in ways that
would break callers of the upstream source.

If you do not call these new functions or make use of the additional APIs, 
this fork should behave identically to the corresponding upstream release.