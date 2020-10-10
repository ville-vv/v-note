

 

## binlog 文件Event协议

### start event 文件开始协议

**v4 format description event**  (size ≥ 91 bytes; the size is 76 + the number of event types):

```
+=====================================+
| event 	  | timestamp         0 : 4    |
| header 	+----------------------------+
|        			 | type_code         4 : 1           | = FORMAT_DESCRIPTION_EVENT = 15
|        			+----------------------------+
|        			| server_id         5 : 4         |
|        			+----------------------------+
|        			| event_length      9 : 4    | >= 91
|        			+----------------------------+
|        			| next_position    13 : 4    |
|        			+----------------------------+
|        			| flags            17 : 2   		 |
+=====================================+
| event 	 | binlog_version   19 : 2    | = 4
| data   	  +----------------------------+
|        			| server_version   21 : 50   |
|        			+----------------------------+
|        			| create_timestamp 71 : 4    |
|        			+----------------------------+
|        			| header_length    75 : 1    |
|        			+----------------------------+
|        			| post-header      76 : n    | = array of n bytes, one byte per event
|        			| lengths for all            |   type that the server knows about
|        			| event types                |
+=====================================+
```

golang 解析头代码示例：

```go
func (h *EventHeader) Decode(data []byte) error {
	if len(data) < EventHeaderSize {
		return errors.Errorf("header size too short %d, must 19", len(data))
	}

	pos := 0

	h.Timestamp = binary.LittleEndian.Uint32(data[pos:])
	pos += 4

	h.EventType = EventType(data[pos])
	pos++

	h.ServerID = binary.LittleEndian.Uint32(data[pos:])
	pos += 4

	h.EventSize = binary.LittleEndian.Uint32(data[pos:])
	pos += 4

	h.LogPos = binary.LittleEndian.Uint32(data[pos:])
	pos += 4

	h.Flags = binary.LittleEndian.Uint16(data[pos:])
	pos += 2

	if h.EventSize < uint32(EventHeaderSize) {
		return errors.Errorf("invalid event size %d, must >= 19", h.EventSize)
	}

	return nil
}
```

### Init HandShate 初始化握手协议

```
1              [0a] protocol version
string[NUL]    server version
4              connection id
string[8]      auth-plugin-data-part-1
1              [00] filler
2              capability flags (lower 2 bytes)
  if more data in the packet:
1              character set
2              status flags
2              capability flags (upper 2 bytes)
  if capabilities & CLIENT_PLUGIN_AUTH {
1              length of auth-plugin-data
  } else {
1              [00]
  }
string[10]     reserved (all [00])
  if capabilities & CLIENT_SECURE_CONNECTION {
string[$len]   auth-plugin-data-part-2 ($len=MAX(13, length of auth-plugin-data - 8))
  if capabilities & CLIENT_PLUGIN_AUTH {
string[NUL]    auth-plugin name
  }
```

#### Fields

- **protocol_version** ([*1*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-1)) -- `0x0a` protocol_version

- **server_version** ([*string.NUL*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.NUL)) -- human-readable server version

- **connection_id** ([*4*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-4)) -- connection id

- **auth_plugin_data_part_1** ([*string.fix_len*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.fix_len)) -- [len=8] first 8 bytes of the auth-plugin data

- **filler_1** ([*1*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-1)) -- `0x00`

- **capability_flag_1** ([*2*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-2)) -- lower 2 bytes of the [`Protocol::CapabilityFlags`](https://dev.mysql.com/doc/internals/en/capability-flags.html#packet-Protocol::CapabilityFlags) (optional)

- **character_set** ([*1*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-1)) -- default server character-set, only the lower 8-bits [`Protocol::CharacterSet`](https://dev.mysql.com/doc/internals/en/character-set.html#packet-Protocol::CharacterSet) (optional)

  This “character set” value is really a collation ID but implies the character set; see the [`Protocol::CharacterSet`](https://dev.mysql.com/doc/internals/en/character-set.html#packet-Protocol::CharacterSet) description.

- **status_flags** ([*2*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-2)) -- [`Protocol::StatusFlags`](https://dev.mysql.com/doc/internals/en/status-flags.html#packet-Protocol::StatusFlags) (optional)

- **capability_flags_2** ([*2*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-2)) -- upper 2 bytes of the [`Protocol::CapabilityFlags`](https://dev.mysql.com/doc/internals/en/capability-flags.html#packet-Protocol::CapabilityFlags)

- **auth_plugin_data_len** ([*1*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-1)) -- length of the combined auth_plugin_data, if auth_plugin_data_len is > 0

- **auth_plugin_name** ([*string.NUL*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.NUL)) -- name of the auth_method that the auth_plugin_data belongs to

### HandShakeResponse

Depending on the servers support for the [`CLIENT_PROTOCOL_41`](https://dev.mysql.com/doc/internals/en/capability-flags.html#flag-CLIENT_PROTOCOL_41) capability and the clients understanding of that flag the client has to send either a [`Protocol::HandshakeResponse41`](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::HandshakeResponse41) or [`Protocol::HandshakeResponse320`](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::HandshakeResponse320).



#### HandshakeResponse41

Handshake Response Packet sent by 4.1+ clients supporting [`CLIENT_PROTOCOL_41`](https://dev.mysql.com/doc/internals/en/capability-flags.html#flag-CLIENT_PROTOCOL_41) capability, if the server announced it in its Initial Handshake Packet. Otherwise (talking to an old server) the [`Protocol::HandshakeResponse320`](https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::HandshakeResponse320) packet must be used.

```
4              capability flags, CLIENT_PROTOCOL_41 always set
4              max-packet size
1              character set
string[23]     reserved (all [0])
string[NUL]    username
  if capabilities & CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA {
lenenc-int     length of auth-response
string[n]      auth-response
  } else if capabilities & CLIENT_SECURE_CONNECTION {
1              length of auth-response
string[n]      auth-response
  } else {
string[NUL]    auth-response
  }
  if capabilities & CLIENT_CONNECT_WITH_DB {
string[NUL]    database
  }
  if capabilities & CLIENT_PLUGIN_AUTH {
string[NUL]    auth plugin name
  }
  if capabilities & CLIENT_CONNECT_ATTRS {
lenenc-int     length of all key-values
lenenc-str     key
lenenc-str     value
   if-more data in 'length of all key-values', more keys and value pairs
  }
```

#### Fields

- **capability_flags** ([*4*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-4)) -- capability flags of the client as defined in [`Protocol::CapabilityFlags`](https://dev.mysql.com/doc/internals/en/capability-flags.html#packet-Protocol::CapabilityFlags)
- **max_packet_size** ([*4*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-4)) -- max size of a command packet that the client wants to send to the server
- **character_set** ([*1*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-1)) -- connection's default character set as defined in [`Protocol::CharacterSet`](https://dev.mysql.com/doc/internals/en/character-set.html#packet-Protocol::CharacterSet).
- **username** ([*string.fix_len*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.fix_len)) -- name of the SQL account which client wants to log in -- this string should be interpreted using the character set indicated by `character set` field.
- **auth-response** ([*string.NUL*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.NUL)) -- opaque authentication response data generated by [*Authentication Method*](https://dev.mysql.com/doc/internals/en/authentication-method.html) indicated by the `plugin name` field.
- **database** ([*string.NUL*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.NUL)) -- initail database for the connection -- this string should be interpreted using the character set indicated by `character set` field.
- **auth plugin name** ([*string.NUL*](https://dev.mysql.com/doc/internals/en/describing-packets.html#type-string.NUL)) -- the [*Authentication Method*](https://dev.mysql.com/doc/internals/en/authentication-method.html) used by the client to generate `auth-response` value in this packet. This is an UTF-8 string.

#### golang 写代码示例

- TODO 待补充











