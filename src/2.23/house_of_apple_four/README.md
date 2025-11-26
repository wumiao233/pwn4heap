## Notice

```C
const struct _IO_jump_t _IO_wfile_jumps =
{
  JUMP_INIT_DUMMY,
  JUMP_INIT(finish, _IO_new_file_finish),
  JUMP_INIT(overflow, (_IO_overflow_t) _IO_wfile_overflow),
  JUMP_INIT(underflow, (_IO_underflow_t) _IO_wfile_underflow),
  JUMP_INIT(uflow, (_IO_underflow_t) _IO_wdefault_uflow),
  JUMP_INIT(pbackfail, (_IO_pbackfail_t) _IO_wdefault_pbackfail),
  JUMP_INIT(xsputn, _IO_wfile_xsputn),
  JUMP_INIT(xsgetn, _IO_file_xsgetn),
  JUMP_INIT(seekoff, _IO_wfile_seekoff),
  JUMP_INIT(seekpos, _IO_default_seekpos),
  JUMP_INIT(setbuf, _IO_new_file_setbuf),
  JUMP_INIT(sync, (_IO_sync_t) _IO_wfile_sync),
  JUMP_INIT(doallocate, _IO_wfile_doallocate),
  JUMP_INIT(read, _IO_file_read),
  JUMP_INIT(write, _IO_new_file_write),
  JUMP_INIT(seek, _IO_file_seek),
  JUMP_INIT(close, _IO_file_close),
  JUMP_INIT(stat, _IO_file_stat),
  JUMP_INIT(showmanyc, _IO_default_showmanyc),
  JUMP_INIT(imbue, _IO_default_imbue)
};
libc_hidden_data_def (_IO_wfile_jumps)

struct _IO_codecvt
{
  void (*__codecvt_destr) (struct _IO_codecvt *);
  enum __codecvt_result (*__codecvt_do_out) (struct _IO_codecvt *,
					     __mbstate_t *,
					     const wchar_t *,
					     const wchar_t *,
					     const wchar_t **, char *,
					     char *, char **);
  enum __codecvt_result (*__codecvt_do_unshift) (struct _IO_codecvt *,
						 __mbstate_t *, char *,
						 char *, char **);
  enum __codecvt_result (*__codecvt_do_in) (struct _IO_codecvt *,
					    __mbstate_t *,
					    const char *, const char *,
					    const char **, wchar_t *,
					    wchar_t *, wchar_t **);
  int (*__codecvt_do_encoding) (struct _IO_codecvt *);
  int (*__codecvt_do_always_noconv) (struct _IO_codecvt *);
  int (*__codecvt_do_length) (struct _IO_codecvt *, __mbstate_t *,
			      const char *, const char *, _IO_size_t);
  int (*__codecvt_do_max_length) (struct _IO_codecvt *);

  _IO_iconv_t __cd_in;
  _IO_iconv_t __cd_out;
};

struct _IO_wide_data
{
  wchar_t *_IO_read_ptr;	/* Current read pointer */
  wchar_t *_IO_read_end;	/* End of get area. */
  wchar_t *_IO_read_base;	/* Start of putback+get area. */
  wchar_t *_IO_write_base;	/* Start of put area. */
  wchar_t *_IO_write_ptr;	/* Current put pointer. */
  wchar_t *_IO_write_end;	/* End of put area. */
  wchar_t *_IO_buf_base;	/* Start of reserve area. */
  wchar_t *_IO_buf_end;		/* End of reserve area. */
  /* The following fields are used to support backing up and undo. */
  wchar_t *_IO_save_base;	/* Pointer to start of non-current get area. */
  wchar_t *_IO_backup_base;	/* Pointer to first valid character of
				   backup area */
  wchar_t *_IO_save_end;	/* Pointer to end of non-current get area. */

  __mbstate_t _IO_state;
  __mbstate_t _IO_last_state;
  struct _IO_codecvt _codecvt;

  wchar_t _shortbuf[1];

  const struct _IO_jump_t *_wide_vtable;
};
```

```C
wint_t
_IO_wfile_underflow (_IO_FILE *fp)
{
  struct _IO_codecvt *cd;
  enum __codecvt_result status;
  _IO_ssize_t count;

  if (__glibc_unlikely (fp->_flags & _IO_NO_READS))
    {
      fp->_flags |= _IO_ERR_SEEN;
      __set_errno (EBADF);
      return WEOF;
    }
  if (fp->_wide_data->_IO_read_ptr < fp->_wide_data->_IO_read_end)
    return *fp->_wide_data->_IO_read_ptr;

  cd = fp->_codecvt;

  /* Maybe there is something left in the external buffer.  */
  if (fp->_IO_read_ptr < fp->_IO_read_end)
    {
      /* There is more in the external.  Convert it.  */
      const char *read_stop = (const char *) fp->_IO_read_ptr;

      fp->_wide_data->_IO_last_state = fp->_wide_data->_IO_state;
      fp->_wide_data->_IO_read_base = fp->_wide_data->_IO_read_ptr =
	fp->_wide_data->_IO_buf_base;
      status = (*cd->__codecvt_do_in) (cd, &fp->_wide_data->_IO_state,
				       fp->_IO_read_ptr, fp->_IO_read_end,
				       &read_stop,
				       fp->_wide_data->_IO_read_ptr,
				       fp->_wide_data->_IO_buf_end,
				       &fp->_wide_data->_IO_read_end);

      fp->_IO_read_base = fp->_IO_read_ptr;
      fp->_IO_read_ptr = (char *) read_stop;

      /* If we managed to generate some text return the next character.  */
      if (fp->_wide_data->_IO_read_ptr < fp->_wide_data->_IO_read_end)
	return *fp->_wide_data->_IO_read_ptr;

      if (status == __codecvt_error)
	{
	  __set_errno (EILSEQ);
	  fp->_flags |= _IO_ERR_SEEN;
	  return WEOF;
	}

      /* Move the remaining content of the read buffer to the beginning.  */
      memmove (fp->_IO_buf_base, fp->_IO_read_ptr,
	       fp->_IO_read_end - fp->_IO_read_ptr);
      fp->_IO_read_end = (fp->_IO_buf_base
			  + (fp->_IO_read_end - fp->_IO_read_ptr));
      fp->_IO_read_base = fp->_IO_read_ptr = fp->_IO_buf_base;
    }
  else
    fp->_IO_read_base = fp->_IO_read_ptr = fp->_IO_read_end =
      fp->_IO_buf_base;

  if (fp->_IO_buf_base == NULL)
    {
      /* Maybe we already have a push back pointer.  */
      if (fp->_IO_save_base != NULL)
	{
	  free (fp->_IO_save_base);
	  fp->_flags &= ~_IO_IN_BACKUP;
	}
      _IO_doallocbuf (fp);

      fp->_IO_read_base = fp->_IO_read_ptr = fp->_IO_read_end =
	fp->_IO_buf_base;
    }

  fp->_IO_write_base = fp->_IO_write_ptr = fp->_IO_write_end =
    fp->_IO_buf_base;

  if (fp->_wide_data->_IO_buf_base == NULL)
    {
      /* Maybe we already have a push back pointer.  */
      if (fp->_wide_data->_IO_save_base != NULL)
	{
	  free (fp->_wide_data->_IO_save_base);
	  fp->_flags &= ~_IO_IN_BACKUP;
	}
      _IO_wdoallocbuf (fp);
    }

  /* Flush all line buffered files before reading. */
  /* FIXME This can/should be moved to genops ?? */
  if (fp->_flags & (_IO_LINE_BUF | _IO_UNBUFFERED))
    {
#if 0
      _IO_flush_all_linebuffered ();
#else
      /* We used to flush all line-buffered stream.  This really isn't
	 required by any standard.  My recollection is that
	 traditional Unix systems did this for stdout.  stderr better
	 not be line buffered.  So we do just that here
	 explicitly.  --drepper */
      _IO_acquire_lock (_IO_stdout);

      if ((_IO_stdout->_flags & (_IO_LINKED | _IO_NO_WRITES | _IO_LINE_BUF))
	  == (_IO_LINKED | _IO_LINE_BUF))
	_IO_OVERFLOW (_IO_stdout, EOF);

      _IO_release_lock (_IO_stdout);
#endif
    }

  _IO_switch_to_get_mode (fp);

  fp->_wide_data->_IO_read_base = fp->_wide_data->_IO_read_ptr =
    fp->_wide_data->_IO_buf_base;
  fp->_wide_data->_IO_read_end = fp->_wide_data->_IO_buf_base;
  fp->_wide_data->_IO_write_base = fp->_wide_data->_IO_write_ptr =
    fp->_wide_data->_IO_write_end = fp->_wide_data->_IO_buf_base;

  const char *read_ptr_copy;
  char accbuf[MB_LEN_MAX];
  size_t naccbuf = 0;
 again:
  count = _IO_SYSREAD (fp, fp->_IO_read_end,
		       fp->_IO_buf_end - fp->_IO_read_end);
  if (count <= 0)
    {
      if (count == 0 && naccbuf == 0)
	{
	  fp->_flags |= _IO_EOF_SEEN;
	  fp->_offset = _IO_pos_BAD;
	}
      else
	fp->_flags |= _IO_ERR_SEEN, count = 0;
    }
  fp->_IO_read_end += count;
  if (count == 0)
    {
      if (naccbuf != 0)
	/* There are some bytes in the external buffer but they don't
	   convert to anything.  */
	__set_errno (EILSEQ);
      return WEOF;
    }
  if (fp->_offset != _IO_pos_BAD)
    _IO_pos_adjust (fp->_offset, count);

  /* Now convert the read input.  */
  fp->_wide_data->_IO_last_state = fp->_wide_data->_IO_state;
  fp->_IO_read_base = fp->_IO_read_ptr;
  const char *from = fp->_IO_read_ptr;
  const char *to = fp->_IO_read_end;
  size_t to_copy = count;
  if (__glibc_unlikely (naccbuf != 0))
    {
      to_copy = MIN (sizeof (accbuf) - naccbuf, count);
      to = __mempcpy (&accbuf[naccbuf], from, to_copy);
      naccbuf += to_copy;
      from = accbuf;
    }
  status = (*cd->__codecvt_do_in) (cd, &fp->_wide_data->_IO_state,
				   from, to, &read_ptr_copy,
				   fp->_wide_data->_IO_read_end,
				   fp->_wide_data->_IO_buf_end,
				   &fp->_wide_data->_IO_read_end);

  if (__glibc_unlikely (naccbuf != 0))
    fp->_IO_read_ptr += MAX (0, read_ptr_copy - &accbuf[naccbuf - to_copy]);
  else
    fp->_IO_read_ptr = (char *) read_ptr_copy;
  if (fp->_wide_data->_IO_read_end == fp->_wide_data->_IO_buf_base)
    {
      if (status == __codecvt_error)
	{
	out_eilseq:
	  __set_errno (EILSEQ);
	  fp->_flags |= _IO_ERR_SEEN;
	  return WEOF;
	}

      /* The read bytes make no complete character.  Try reading again.  */
      assert (status == __codecvt_partial);

      if (naccbuf == 0)
	{
	  if (fp->_IO_read_base < fp->_IO_read_ptr)
	    {
	      /* Partially used the buffer for some input data that
		 produces no output.  */
	      size_t avail = fp->_IO_read_end - fp->_IO_read_ptr;
	      memmove (fp->_IO_read_base, fp->_IO_read_ptr, avail);
	      fp->_IO_read_ptr = fp->_IO_read_base;
	      fp->_IO_read_end -= avail;
	      goto again;
	    }
	  naccbuf = fp->_IO_read_end - fp->_IO_read_ptr;
	  if (naccbuf >= sizeof (accbuf))
	    goto out_eilseq;

	  memcpy (accbuf, fp->_IO_read_ptr, naccbuf);
	}
      else
	{
	  size_t used = read_ptr_copy - accbuf;
	  if (used > 0)
	    {
	      memmove (accbuf, read_ptr_copy, naccbuf - used);
	      naccbuf -= used;
	    }

	  if (naccbuf == sizeof (accbuf))
	    goto out_eilseq;
	}

      fp->_IO_read_ptr = fp->_IO_read_end = fp->_IO_read_base;

      goto again;
    }

  return *fp->_wide_data->_IO_read_ptr;
}
libc_hidden_def (_IO_wfile_underflow)
```
