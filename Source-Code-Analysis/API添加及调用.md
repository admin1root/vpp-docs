# API新增及调用

以flowprobe_rx_interface_add_del为例说明API的添加及调用过程

## 在api文件中定义通信结构体

\*.api中定义vat与vpp通信的结构体，然后由vppapigen文件处理，最终生成一系列的头文件。两边都包含这些头文件，这样vat与vpp就使用了相同的结构体通信了。 

在flowprobe.api中添加如下代码：
```
/** \brief Enable / disable per-packet IPFIX recording on an interface
    @param client_index - opaque cookie to identify the sender
    @param context - sender context, to match reply w/ request
    @param is_add - add address if non-zero, else delete
    @param which - flags indicating forwarding path
    @param sw_if_index - index of the interface
*/
autoreply manual_print define flowprobe_rx_interface_add_del
{
  /* Client identifier, set from api_main.my_client_index */
  u32 client_index;

  /* Arbitrary context, so client can match reply to request */
  u32 context;

  /* Enable / disable the feature */
  bool is_add;
  vl_api_flowprobe_which_flags_t which;

  /* Interface handle */
  vl_api_interface_index_t sw_if_index;
  option vat_help = "<intfc> [disable]";
};
```

在flowprobe.api_type.h中生成了vl_api_flowprobe_rx_interface_add_del_t,vl_api_flowprobe_rx_interface_add_del_reply_t，供vpp使用，如下：
```
typedef struct __attribute__ ((packed)) _vl_api_flowprobe_rx_interface_add_del {
    u16 _vl_msg_id;
    u32 client_index;
    u32 context;
    bool is_add;
    vl_api_flowprobe_which_flags_t which;
    vl_api_interface_index_t sw_if_index;
} vl_api_flowprobe_rx_interface_add_del_t;
typedef struct __attribute__ ((packed)) _vl_api_flowprobe_rx_interface_add_del_reply {
    u16 _vl_msg_id;
    u32 context;
    i32 retval;
} vl_api_flowprobe_rx_interface_add_del_reply_t;
```

在flowprobe.api.vapi.h中生成了两个消息结构，供client使用，包括请求消息体和响应消息体，如下：
```
\\请求消息体
typedef struct __attribute__ ((__packed__)) {
  bool is_add;
  vapi_enum_flowprobe_which_flags which;
  vapi_type_interface_index sw_if_index; 
} vapi_payload_flowprobe_rx_interface_add_del;

typedef struct __attribute__ ((__packed__)) {
  vapi_type_msg_header2_t header;
  vapi_payload_flowprobe_rx_interface_add_del payload;
} vapi_msg_flowprobe_rx_interface_add_del;

\\响应消息体
typedef struct __attribute__ ((__packed__)) {
  i32 retval; 
} vapi_payload_flowprobe_rx_interface_add_del_reply;

typedef struct __attribute__ ((__packed__)) {
  vapi_type_msg_header1_t header;
  vapi_payload_flowprobe_rx_interface_add_del_reply payload;
} vapi_msg_flowprobe_rx_interface_add_del_reply;
```

同时，flowprobe.api.vapi.h还生成了一系列的函数，供client使用，包括消息构造函数，字节序转换函数，消息处理函数等，如下：
```
\\消息构造
static inline vapi_msg_flowprobe_rx_interface_add_del* vapi_alloc_flowprobe_rx_interface_add_del(struct vapi_ctx_s *ctx)
{
  vapi_msg_flowprobe_rx_interface_add_del *msg = NULL;
  const size_t size = sizeof(vapi_msg_flowprobe_rx_interface_add_del);
  /* cast here required to play nicely with C++ world ... */
  msg = (vapi_msg_flowprobe_rx_interface_add_del*)vapi_msg_alloc(ctx, size);
  if (!msg) {
    return NULL;
  }
  msg->header.client_index = vapi_get_client_index(ctx);
  msg->header.context = 0;
  msg->header._vl_msg_id = vapi_lookup_vl_msg_id(ctx, vapi_msg_id_flowprobe_rx_interface_add_del);

  return msg;
}

\\字节序转换
static inline void vapi_msg_flowprobe_rx_interface_add_del_payload_hton(vapi_payload_flowprobe_rx_interface_add_del *payload)
{
  payload->sw_if_index = htobe32(payload->sw_if_index);
}

static inline void vapi_msg_flowprobe_rx_interface_add_del_payload_ntoh(vapi_payload_flowprobe_rx_interface_add_del *payload)
{
  payload->sw_if_index = be32toh(payload->sw_if_index);
}

static inline void vapi_msg_flowprobe_rx_interface_add_del_hton(vapi_msg_flowprobe_rx_interface_add_del *msg)
{
  VAPI_DBG("Swapping `vapi_msg_flowprobe_rx_interface_add_del'@%p to big endian", msg);
  vapi_type_msg_header2_t_hton(&msg->header);
  vapi_msg_flowprobe_rx_interface_add_del_payload_hton(&msg->payload);
}

static inline void vapi_msg_flowprobe_rx_interface_add_del_ntoh(vapi_msg_flowprobe_rx_interface_add_del *msg)
{
  VAPI_DBG("Swapping `vapi_msg_flowprobe_rx_interface_add_del'@%p to host byte order", msg);
  vapi_type_msg_header2_t_ntoh(&msg->header);
  vapi_msg_flowprobe_rx_interface_add_del_payload_ntoh(&msg->payload);
}

\\消息处理函数
static inline vapi_error_e vapi_flowprobe_rx_interface_add_del(struct vapi_ctx_s *ctx,
  vapi_msg_flowprobe_rx_interface_add_del *msg,
  vapi_error_e (*callback)(struct vapi_ctx_s *ctx,
                           void *callback_ctx,
                           vapi_error_e rv,
                           bool is_last,
                           vapi_payload_flowprobe_rx_interface_add_del_reply *reply),
  void *callback_ctx)
{
  if (!msg || !callback) {
    return VAPI_EINVAL;
  }
  if (vapi_is_nonblocking(ctx) && vapi_requests_full(ctx)) {
    return VAPI_EAGAIN;
  }
  vapi_error_e rv;
  if (VAPI_OK != (rv = vapi_producer_lock (ctx))) {
    return rv;
  }
  u32 req_context = vapi_gen_req_context(ctx);
  msg->header.context = req_context;
  vapi_msg_flowprobe_rx_interface_add_del_hton(msg);
  if (VAPI_OK == (rv = vapi_send (ctx, msg))) {
    vapi_store_request(ctx, req_context, false, (vapi_cb_t)callback, callback_ctx);
    if (VAPI_OK != vapi_producer_unlock (ctx)) {
      abort (); /* this really shouldn't happen */
    }
    if (vapi_is_nonblocking(ctx)) {
      rv = VAPI_OK;
    } else {
      rv = vapi_dispatch(ctx);
    }
  } else {
    vapi_msg_flowprobe_rx_interface_add_del_ntoh(msg);
    if (VAPI_OK != vapi_producer_unlock (ctx)) {
      abort (); /* this really shouldn't happen */
    }
  }
  return rv;
}
```

\*.api.vapi.h就是我们通常所说的C high-level API，C++ high-level API在\*.api.vapi.hpp中，这里就不详细列举了。
client 可以调用响应的high-level API与vpp通信，发送请求消息给vpp并通过回调处理vpp的响应消息。
通过check框架调用代码示例如下：
```
vapi_error_e flowprobe_rx_interface_add_del_cb(struct vapi_ctx_s *ctx,
                           void *callback_ctx,
                           vapi_error_e rv,
                           bool is_last,
                           vapi_payload_flowprobe_rx_interface_add_del_reply *reply)
{
	  ck_assert_int_eq (VAPI_OK, rv);
	  ck_assert_int_eq (true, is_last);
	  ck_assert_str_eq (VAPI_OK,reply->retval);
	  ++*(int *) caller_ctx;
	  return VAPI_OK;

}


START_TEST (test_flowprobe_rx_interface_add_del)
{
	int called = 0;
	printf("--- flowprobe_rx_interface_add_del via blocking callback API ---", port);

	vapi_msg_flowprobe_rx_interface_add_del *friad = vapi_alloc_flowprobe_rx_interface_add_del(ctx);
	friad->payload.is_add = 1;
	friad->payload.which = FLOWPROBE_WHICH_FLAG_IP4;
	friad->payload.sw_if_index = 3;
	ck_assert_ptr_ne (NULL, sv);
	vapi_error_e rv = vapi_flowprobe_rx_interface_add_del(ctx, friad,  callback, called);
	ck_assert_int_eq (VAPI_OK, rv);
	ck_assert_int_eq (1, called);
}
```

**关键字说明**
+ 关键字 autoreply：用于自动生成回复消息结构
+ 关键字manual_print：xxx_print函数是用来打印消息结构体内容的。默认情况下会自动生成。添加该关键字后需手动实现。
+ 关键字define：define 关键字是转化的关键。每个定义都要加上。

## 添加vpp处理函数

client与vpp通信，发送消息给vpp后，vpp会对消息进行处理。
vpp中注册了一个vl_api_clnt_node节点，该节点的处理函数vl_api_clnt_process中调用了vl_mem_api_handle_msg_main来处理api消息，vl_mem_api_handle_msg_main中调用了消息响应的处理函数vl_api_XXX_t_handler，vl_api_XXX_t_handler需自行实现。

在flowprobe中，我们在flowprobe.c文件中实现了消息的处理：
```
/**
 * @brief API message handler
 * @param mp vl_api_flowprobe_rx_interface_add_del_t * mp the api message
 */
void vl_api_flowprobe_rx_interface_add_del_t_handler
  (vl_api_flowprobe_rx_interface_add_del_t * mp)
{
  flowprobe_main_t *fm = &flowprobe_main;
  vl_api_flowprobe_rx_interface_add_del_reply_t *rmp;
  u32 sw_if_index = ntohl (mp->sw_if_index);
  int rv = 0;

  VALIDATE_SW_IF_INDEX (mp);

  if (fm->record == 0)
    {
      clib_warning ("Please specify flowprobe params record first...");
      rv = VNET_API_ERROR_CANNOT_ENABLE_DISABLE_FEATURE;
      goto out;
    }

  rv = validate_feature_on_interface (fm, sw_if_index, mp->which);
  if ((rv == 1 && mp->is_add == 1) || rv == 0)
    {
      rv = VNET_API_ERROR_CANNOT_ENABLE_DISABLE_FEATURE;
      goto out;
    }

  rv = flowprobe_rx_interface_add_del_feature
    (fm, sw_if_index, mp->which, mp->is_add);

out:
  BAD_SW_IF_INDEX_LABEL;

  REPLY_MACRO (VL_API_FLOWPROBE_RX_INTERFACE_ADD_DEL_REPLY);
}
```
消息处理的结束位置，通常调用REPLY_MACRO发送回复消息。

另外，我们在定义通信结构体时，使用了关键字manual_print，所以需要自行实现一个vl_api_XXX_t_print。
在flowprove.c中实现vl_api_flowprobe_rx_interface_add_del_t_print，如下：
```
/**
 * @brief API message custom-dump function
 * @param mp vl_api_flowprobe_rx_interface_add_del_t * mp the api message
 * @param handle void * print function handle
 * @returns u8 * output string
 */
static void *vl_api_flowprobe_rx_interface_add_del_t_print
  (vl_api_flowprobe_rx_interface_add_del_t * mp, void *handle)
{
  u8 *s;

  s = format (0, "SCRIPT: flowprobe_rx_interface_add_del ");
  s = format (s, "sw_if_index %d is_add %d which %d ",
	      clib_host_to_net_u32 (mp->sw_if_index),
	      (int) mp->is_add, (int) mp->which);
  FINISH;
}
```

## vat与vpp通信的实现
在*_test.c中添加api_XXX函数，在函数中实现消息构造及与vpp的通信。

在flowprobe_test.c中实现如下：
```
static int
api_flowprobe_rx_interface_add_del (vat_main_t * vam)
{
  unformat_input_t *i = vam->input;
  int enable_disable = 1;
  u8 which = FLOW_VARIANT_IP4;
  u32 sw_if_index = ~0;
  vl_api_flowprobe_rx_interface_add_del_t *mp;
  int ret;

  /* Parse args required to build the message */
  while (unformat_check_input (i) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (i, "%U", unformat_sw_if_index, vam, &sw_if_index))
	;
      else if (unformat (i, "sw_if_index %d", &sw_if_index))
	;
      else if (unformat (i, "disable"))
	enable_disable = 0;
      else if (unformat (i, "ip4"))
	which = FLOW_VARIANT_IP4;
      else if (unformat (i, "ip6"))
	which = FLOW_VARIANT_IP6;
      else if (unformat (i, "l2"))
	which = FLOW_VARIANT_L2;
      else
	break;
    }

  if (sw_if_index == ~0)
    {
      errmsg ("missing interface name / explicit sw_if_index number \n");
      return -99;
    }

  /* Construct the API message */
  M (FLOWPROBE_RX_INTERFACE_ADD_DEL, mp);
  mp->sw_if_index = ntohl (sw_if_index);
  mp->is_add = enable_disable;
  mp->which = which;

  /* send it... */
  S (mp);

  /* Wait for a reply... */
  W (ret);
  return ret;
}
```
还可以添加一个vl_api_XXX_reply_t_handler的函数，来处理vpp回复的消息。
由于flowprobe没必要处理，所以没有添加。
