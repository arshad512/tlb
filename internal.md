110

# Lustre Gems! (Work in progress....)

## Lustre programming (Lustre API)

### What is a transaction
this does not support mulitple transactions yet, only 1 update RPC each time betwee MDTs
 

## Linux Kernel programming (Kernel API used under Lustre)

- ### KIOV vs Kvec (https://review.whamcloud.com/#/c/36826/)


LU-13004 ptlrpc: Allow BULK_BUF_KIOV to accept a kvec

Bulk descriptor of type PTLRPC_BULK_BUF_KIOV are comprised
of a list of page+offset+len.
If the calling code actually has a virtual-address+len, it
cannot current use BULK_BUF_KIOV and must use BULK_BUF_KVEC.

However it is quite easy to convert virtual-address+len
to a list of page+offset+len.

So we can add a ->add_iov_frag interface for KIOV descriptors, and
then we will be able to use KIOV descriptors for everything.  The
caller must ensure to allocate a large enough descriptor, taking
into account the size of each exptected kvec.

No, that wouldn't be a good idea.
It is only reasonable to "pin" a page if we "own" it already.
When we have a virtual address plus length, we might not own the page.
memory provided by kmalloc is a good example.  mapping the address
to a struct page, and then incrementing the reference on that page probably won't do what you want.  It certainly won't stop the page from being reused by the slab allocater.


const struct ptlrpc_bulk_frag_ops ptlrpc_bulk_kiov_nopin_ops = {
	.add_kiov_frag	= ptlrpc_prep_bulk_page_nopin,
	.release_frags	= ptlrpc_release_bulk_noop,
	.add_iov_frag	= ptlrpc_prep_bulk_frag_pages,
};
EXPORT_SYMBOL(ptlrpc_bulk_kiov_nopin_ops);

=>
LU-13004 target: use KIOV for out_handle

Convert out_handle() use use a BULK_BUF_KIOV rather than
a BULK_BUF_KVEC.

This is a step towards removed KVEC support and standardizing
on KIOV.

