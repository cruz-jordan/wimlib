# Additional APIs in this fork

This fork exposes one extra public helper for accessing a WIM's XML metadata
in a managed-friendly way. Plus a corresponding free function:

## [wimlib_get_wim_xml](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L3354)
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

## [wimlib_free_tstr](https://github.com/cruz-jordan/wimlib/blob/master/include/wimlib.h#L3217)
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
	if(tstr)
		FREE(tstr);
}
```

# Rationale

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

# Behaviour and compatability

- No existing functions or structures have been removed or modified.
- `wimlib_get_xml_data()` remains available and behaves as in upstream.
- Error codes returned by `wimlib_get_wim_xml()` are standard `wimlib_error_code`
values consistent with the related XML functions.

If you do not call these new functions, this fork should behave identically
to the corresponding upstream release.