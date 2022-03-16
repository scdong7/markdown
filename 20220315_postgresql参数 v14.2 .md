# postgresql参数 v14.2 

## 1.系统参数

### 1.1 配置文件

1. postgresql.conf

2. postgresql.auto.conf（保存通过ALTER SYSTEM命令提供的设置）

- pg_ctl reload或 调用 SQL 函数pg_reload_conf()来发送SIGHUP信号)后都会重新读取这个配置文件。

- 系统视图pg_file_settings 可以有助于对配置文件中的更改进行提前测试，或者在SIGHUP 信号没有达到预期效果时用来诊断问题。

### 1.2 通过SQL影响参数

1. alter system 改变全局

2. alter database 针对一个数据库

3. alter role 命令允许用用户指定的值来覆盖全局设置和数据库设置。
4. set 允许修改对于一个会话可以本地设置的参数的当前值
   - 系统视图pg_settings可以被用来查看和改变 会话本地的值,并可以在视图上执行update更新。

### 1.3 通过shell 

1. 在服务器启动期间，可以通过-c命令行参数把参数设置传递给 postgres命令。

   例如:

   postgres -c log_connections=yes -c log_destination='syslog'

   这种方式提供的设置会覆盖通过postgresql.conf或者 ALTER SYSTEM提供的设置，

   因此除 了重启服务器之外无法从全局上改变它们。

2. 当通过libpq启动一个客户端会话时，可以使用PGOPTIONS 环境变量指定参数设置。

   这种 方式建立的设置构成了会话生存期间的默认值，但是不会影响 其他的会话。

   由于历史原 因，PGOPTIONS的格式和启动 postgres命令时用到的相似，特别是-c标志必须被指定。 例 如:

   env PGOPTIONS="-c geqo=off -c statement_timeout=5min" psql

## 2.参数详解

### 2.1 文件位置

```
# -----------------------------
# PostgreSQL configuration file
# -----------------------------
#
# This file consists of lines of the form:
#
#   name = value
#
# (The "=" is optional.)  Whitespace may be used.  Comments are introduced with
# "#" anywhere on a line.  The complete list of parameter names and allowed
# values can be found in the PostgreSQL documentation.
#
# The commented-out settings shown in this file represent the default values.
# Re-commenting a setting is NOT sufficient to revert it to the default value;
# you need to reload the server.
#
# This file is read on server startup and when the server receives a SIGHUP
# signal.  If you edit the file on a running system, you have to SIGHUP the
# server for the changes to take effect, run "pg_ctl reload", or execute
# "SELECT pg_reload_conf()".  Some parameters, which are marked below,
# require a server shutdown and restart to take effect.
#
# Any parameter can also be given as a command-line option to the server, e.g.,
# "postgres -c log_connections=on".  Some parameters can be changed at run time
# with the "SET" SQL command.
#
# Memory units:  B  = bytes            Time units:  us  = microseconds
#                kB = kilobytes                     ms  = milliseconds
#                MB = megabytes                     s   = seconds
#                GB = gigabytes                     min = minutes
#                TB = terabytes                     h   = hours
#                                                   d   = days


#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------

# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.

#data_directory = 'ConfigDir'		# use data in another directory
																# (change requires restart)
#hba_file = 'ConfigDir/pg_hba.conf'	# host-based authentication file
																		# (change requires restart)
#ident_file = 'ConfigDir/pg_ident.conf'	# ident configuration file
																				# (change requires restart)

# If external_pid_file is not explicitly set, no extra PID file is written.
#external_pid_file = ''			# write an extra PID file
														# (change requires restart)
```

### 2.2 连接和认证

#### 2.2.1 连接设置

```
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -
###连接设置
listen_addresses = '*'		# what IP address(es) to listen on;
                          # comma-separated list of addresses;
                          # defaults to 'localhost'; use '*' for all
                          # (change requires restart)
#port = 5432							# (change requires restart)
max_connections = 100			# (change requires restart)
#决定数据库的最大并发连接数。默认值通常是 100 个连接，但是如果内核设置不支持 (initdb时决定)，可能会比这个 数少。
#这个参数只能在服务器启动时设置。
#当运行一个后备服务器时，你必须设置这个参数等于或大于主服务器上的参数。否则， 后备服务器上可能无法允许查询。

#superuser_reserved_connections = 3	# (change requires restart)
#决定为PostgreSQL超级用户连接而保留的连接“槽”数。 同时活跃的并发连接最 多max_connections个。
#任何时候，活跃的并发连接数最多为max_connections减去 superuser_reserved_connections，
#新连接就只能由超级用户发起了，并且不会有新的 复制连接被接受。
#默认值是 3 连接 。这个值必须小于max_connections。 这个参数只能在服务器启动时 设置。

#unix_socket_directories = '/tmp'	# comma-separated list of directories
																	# (change requires restart)
#指定服务器用于监听来自客户端应用的连接的 Unix 域套接字目录。通过列出用逗号分 隔的多个目录可以建立多个套接字。项之间的空白被忽略，如#果你需要在名字中包括空 白或逗号，在目录名周围放上双引号。一个空值指定在任何 Unix 域套接字上都不监 听，在这种情况中只能使用 TCP/IP #套接字来连接到服务器。默认值通常是/tmp，但是 在编译时可以被改变。这个参数只能在服务器启动时设置。
#除了套接字文件本身(名为.s.PGSQL.nnnn，其中nnnn是服务器的端口号)，一个名 为.s.PGSQL.nnnn.lock的普通文件会在每一个#unix_socket_directories目录中被创建。 任何一个都不应该被手工移除。
#Windows下没有 Unix 域套接字，因此这个参数与 Windows 无关。

#unix_socket_group = ''			# (change requires restart)
#设置 Unix 域套接字的所属组(套接字的所属用户总是启动服务器的用户)。可以与选 项unix_socket_permissions一起用于对 Unix域连接进行#访问控制。默认是一个空字符 串，表示服务器用户的默认组。这个参数只能在服务器启动时设置。
#Windows 下没有 Unix 域套接字，因此这个参数与 Windows 无关。


#unix_socket_permissions = 0777		# begin with 0 to use octal notation
																	# (change requires restart)
#设置 Unix 域套接字的访问权限。Unix 域套接字使用普通的 Unix 文件系统权限集。这 个参数值应该是数字的形式，也就是系统调用chmod和#umask接受的 形式(如果使用自定 义的八进制格式，数字必须以一个0(零)开头)。
#默认的权限是0777，意思是任何人都可以连接。合理的候选是0770(只有用户和同组的 人可以访问， 又见unix_socket_group)和0700(只有用户#自己可以访问)(请注意， 对于 Unix 域套接字，只有写权限有麻烦，因此没有对读取和执行权限的设置和收 回)。 
#这个参数只能在服务器启动时设置。
#这个参数与完全忽略套接字权限的系统无关，尤其是自版本10以上的Solaris。 在那些 系统上，可以通过把unix_socket_directories指向一个#把搜索权限 限制给指定用户的 目录来实现相似的效果。 因为 Windows 下没有 Unix 域套接字，因此这个参数也与 Windows 无关。


#bonjour = off				# advertise server via Bonjour
											# (change requires restart)
#通过Bonjour广告服务器的存在。默认值是关闭。 这个参数只能在服务器启动时设置。
#Bonjour是苹果为基于组播域名服务(multicast DNS)的开放性零设置网络标准所起的名字，能自动发现IP网络上的电脑、设备和服务。
#Bonjour使用工业标准的IP协议来允许设备自动发现彼此，而不需输入IP地址或配置DNS 服务器。


#bonjour_name = ''			# defaults to the computer name
												# (change requires restart)
#指定Bonjour服务名称。空字符串''(默认值)表示使用计算机名。 
#如果编译时没有打 开Bonjour支持那么将忽略这个参数。这个参数只能在服务器启动时设置。

# - TCP settings -
# see "man tcp" for details

#tcp_keepalives_idle = 0				# TCP_KEEPIDLE, in seconds;
																# 0 selects the system default
#规定在操作系统向客户端发送一个TCP keepalive消息后无网络活动的时间总量。 如果 指定值时没有单位，则以秒为单位。
#值0(默认值)表示选择操作系统默认值。 指定不活 动多少秒之后通过 TCP 向客户端发送一个 keepalive 消息。 
#0值表示使用默认值。 这 个参数只有在支持TCP_KEEPIDLE或等效套接字选项的系统或 Windows 上才可以使用。
#在其他系统上，它必须为零。在通过 Unix 域套接字连接的会话中，这个参数被忽略并且 总是读作零。				
#在Windows 上，设定值为0将设置这个参数为 2 小时，因为 Windows 不 支持读取系统默认值。


#tcp_keepalives_interval = 0		# TCP_KEEPINTVL, in seconds;
																# 0 selects the system default
#规定未被客户端确认收到的TCP keepalive消息应重新传输的时间长度。 如果指定值时 没有单位，则以秒为单位。
#值0(默认值)表示选择操作系统默认值。 这个参数只有在支 持TCP_KEEPINTVL或等效套接字选项的系统或 Windows 上才可以使用。
#在其他系统上， 必须为零。在通过 Unix域套接字连接的会话中，这个参数被忽略并总被读作零。		
#在 Windows 上，设定值为0将设置这个参数为 1 秒，因为 Windows 不支 持读取系统默认值。


#tcp_keepalives_count = 0				# TCP_KEEPCNT;
																# 0 selects the system default
#指定服务器到客户端的连接被认为中断之前可以丢失的TCP keepalive消息的数量。
#值 0(默认值)表示选择操作系统默认值。 这个参数只有在支持TCP_KEEPCNT或等效套接字选 项的系统上才可以使用。
#在其他系统上，必须为零。在通过 Unix 域套接字连接的会话 中，这个参数被忽略并总被读作零。			
#Windows 不支持该参数，且必须为零。

#tcp_user_timeout = 0						# TCP_USER_TIMEOUT, in milliseconds;
																# 0 selects the system default
																
#指定传输的数据在TCP连接被强制关闭之前可以保持未确认状态的时间量。 如果指定 值时没有单位，则以毫秒为单位。
#值0(默认值)表示选择操作系统默认值。 这个参数只 有在支持TCP_USER_TIMEOUT的系统上才被支持;
#在其他系统上，它必须为零。 在通过 Unix-domain 套接字连接的会话中，此参数将被忽略并且始终读取为零。
#在Windows上不支持该参数，并且必须为零。 

#client_connection_check_interval = 0	# time between checks for client
																			# disconnection while running queries;
																			# 0 for never
#设置运行查询时可选检查客户端是否仍处于连接状态之间的时间间隔。
#检查是通过轮询套接字来执行的，如果内核报告连接已关闭，那么长时间运行的查询可以更快地中止。
#此选项当前仅在支持轮询系统调用的非标准POLLRDHUP扩展的系统上可用，包括Linux。
#如果指定的值没有单位，则以毫秒为单位。默认值为0，这将禁用连接检查。
#在没有连接检查的情况下，服务器将仅在下次与套接字交互时，即在等待、接收或发送数据时检测到连接丢失。
#为了使内核本身能够在包括网络故障在内的所有情况下可靠地在已知时间范围内检测丢失的TCP连接，
#可能还需要调整操作系统的TCP keepalive设置，
#或者调整PostgreSQL的TCP_keepalives_idle、TCP_keepalives_interval和TCP_keepalives_count设置。
```

#### 2.2.2 安全和认证

```
# - Authentication -
#认证

#authentication_timeout = 1min		# 1s-600s

#允许完成客户端认证的最长时间。如果一个客户端没有在这段时间里完成认证协议，服 务器将关闭连接。 
#这样就避免了出问题的客户端无限制地占有一个连接。如果指定值时 没有单位，则以秒为单位。 默认值是 1分钟(1m)。
#这个参数只能在服务器命令行上或 者在postgresql.conf文件中设置

#password_encryption = scram-sha-256	# scram-sha-256 or md5

#当在CREATE ROLE或者ALTER ROLE中指定了口令时，这个参数决定用于加密该口令的算 法。
#注意老的客户端可能缺少对SCRAM认证机制的支持，因此无法使用用SCRAM-SHA-256加密 的口令。

#db_user_namespace = off

#这个参数启用针对每个数据库的用户名。这个参数默认是关掉的。
#这个参数只能 在postgresql.conf文件中或者服务器命令行上设置。
#如果这个参数为打开，应该把用户创建成username@dbname的形式。
#当一个连接客户端传 来username时，@和数据库名会被追加到用户名并且服务器会查找这个与数据库相关的用 户名。
#注意在SQL环境中用含有@的名称创建用户时，需要把用户名放在引号内。
#在这个参数被启用时，仍然可以创建平常的全局用户。而在客户端中指定这种用户时只 需要简单地追加@，例如joe@。
#在服务器查找该用户名之前，@会被剥离掉。
#db_user_namespace会导致客户端和服务器的用户名表达形式不同。认证检查总是会以服 务器的用户名表达形式来完成，
#因此认证方法必须针对服务器用户名而不是客户端用户 名来配置。
#由于md5方法在客户端和服务器两端都使用用户名作为salt，md5不能 与db_user_namespace同时使用。

# GSSAPI using Kerberos
#krb_server_keyfile = 'FILE:${sysconfdir}/krb5.keytab'

#设置Kerberos服务器密钥文件的位置。

#krb_caseins_users = off
#设置是否应该以大小写不敏感的方式对待GSSAPI用户名。默认值是off(大小写敏感)
```

#### 2.2.3 SSL

```
# - SSL -

#ssl = off
#启用SSL连接。

#ssl_ca_file = ''

#指定包含 SSL 服务器证书颁发机构(CA)的文件名。相对路径是相对于数据目录的。
#这个参数只能在postgresql.conf文件中或者服务器命令行上设置。默认值为空，表示没有载入CA文件，并且客户端证书验证没有被执行。

#ssl_cert_file = 'server.crt'
#指定包含 SSL 服务器证书的文件名。相对路径是相对于数据目录的。
#这个参数只能 在postgresql.conf文件中或者服务器命令行上设置。默认值是server.crt。

#ssl_crl_file = ''

#指定包含 SSL 服务器证书撤销列表(CRL)的文件名。其中的相对路径是相对于数据目 录的。
#这个参数只能在postgresql.conf文件中或者服务器命令行上设置。默认值是空， 表示没有载入CRL文件。

#ssl_crl_dir = ''
#ssl_key_file = 'server.key'

#指定包含 SSL 服务器私钥的文件名。相对路径是相对于数据目录。
#这个参数只能 在postgresql.conf文件中或者服务器命令行上设置。默认值是server.key。

#ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL' # allowed SSL ciphers

#指定一个SSL密码列表，用于安全连接。这个设置的语法和所支持的值列表可以 参见 OpenSSL包中的 ciphers手册页。
#这个参数只能在postgresql.conf文件中或者服务器命 令行上设置。默认值是 HIGH:MEDIUM:+3DES:!aNULL。
#默认值通常是一种合理的选择，除 非用户有特定的安全性需求。

#ssl_prefer_server_ciphers = on

#指定是否使用服务器的 SSL 密码首选项，而不是用客户端的。
#这个参数只能 在postgresql.conf文件中或者服务器命令行上设置。 默认值是 on。
#老的PostgreSQL版本没有这个设置并且总是使用客户端的首选项。
#这个设置主要用于与 那些版本 的向后兼容性。使用服务器的首选项通常会更好，因为服务器更可能会被合适 地配置。

#ssl_ecdh_curve = 'prime256v1'

#指定用在ECDH密钥交换中的曲线名称。它需要被所有连接的客户端支持。 它不需要与服 务器椭圆曲线密钥使用的曲线相同。
#这个参数只能在postgresql.conf文件中或者服务器 命令行上设置。默认值是prime256v1。
#OpenSSL 命名了最常见的曲线: prime256v1 (NIST P-256)、 secp384r1 (NIST P-384)、 secp521r1 (NIST P-521)。
#openssl ecparam -list_curves命令可以显示可用曲线的完 整列表。不过并不是所有的都在TLS中可用。

#ssl_min_protocol_version = 'TLSv1.2'
#设置要使用的最小SSL/TLS协议版本。当前的可用版本包括: TLSv1, TLSv1.1, TLSv1.2, TLSv1.3. 旧版本的 OpenSSL 库不支持所有值;
#如果选择了不支持的设置将会引发错误。 TLS 1.0之前的协议版本，也就是SSL 版本 2 and 3，总是禁用的。
#默认值是 TLSv1, 主要用来支持 OpenSSL 库的老版本。 如果所有软件组件能支持较新 的协议版本，您或许可以将它设置为较高的值。

#ssl_max_protocol_version = ''

#设定要使用的最大SSL/TLS协议版本。 有效的版本为 ssl_min_protocol_version, 添加 一个空字符串，允许任何协议版本。 
#默认为允许任何版本。设置最大协议版本主要用于 测试，或者某个组件在与较新的协议配合工作时出现了问题。

#ssl_dh_params_file = ''

#指定含有用于SSL密码的所谓临时DH家族的Diffie-Hellman参数的文件名。默认值为空， 这种情况下将使用内置的默认DH参数。
#使用自定义的DH参数可以降低攻击者破解众所周 知的内置DH参数的风险。
#可以用命令openssl dhparam -out dhparams.pem 2048创建自 己的DH参数文件。
#这个参数只能在 postgresql.conf文件中或服务器命令行上进行设置。

#ssl_passphrase_command = ''

#设置当需要一个密码(例如一个私钥)来解密SSL文件时会调用的一个外部命令。默认情 况下，这个参数为空，表示使用内建的提示机制。
#该命令必须将密码打印到标准输出并且以代码0退出。在该参数值中，%p被替换为一个提 示字符串(要得到文字%，应该写成%%)。注意该提示字符串将#可能含有空格，因此要确 保加上适当的引号。如果输出的末尾有单一的新行，它会被剥离掉。
#该命令实际上并不一定要提示用户输入一个密码。它可以从文件中读取密码、从钥匙链 得到密码等等。确保选中的机制足够安全是用户的责任。
#这个参数只能在 postgresql.conf文件中或服务器命令行上进行设置。

#ssl_passphrase_command_supports_reload = off

#这个参数决定在配置重载期间如果一个密钥文件需要口令时，是否也调 用ssl_passphrase_command设置的密码命令。 
#如果这个参数为off(默认)，那么在重 载期间将忽略ssl_passphrase_command，
#如果在此期间需要密码则SSL配置将不会被重 载。 对于要求一个TTY(当服务器正在运行时可能是不可用的)来进行提示的命令，这 种设置是合适的。 #例如，如果密码是从一个文件中得到的，将这个参数设置为on可能是 合适的。
#这个参数只能在postgresql.conf文件中或者服务器命令行上设置。


```

### 2.3 资源消耗

#### 2.3.1 内存

```
#------------------------------------------------------------------------------
# RESOURCE USAGE (except WAL)
#------------------------------------------------------------------------------

# - Memory -

shared_buffers = 128MB			# min 128kB
														# (change requires restart)
														
#设置数据库服务器将使用的共享内存缓冲区量。默认通常是128MB，但是如果你的内核设置不支持(在initdb时决定)，那么可以会更少。 
#这个设置必须至少 为 128 千字节。不过为了更好的性能，通常会使用明显高于最小值的设置。 
#如果指定 值时没有单位，则以块为单位，即BLCKSZ字节，通常为8kB.(BLCKSZ 的非默认值改变最 小值。) 此参数只能在服务器启动时设置。

#如果有一个专用的 1GB 或更多内存的数据库服务器，一个合理的shared_buffers开 始值是系统内存的 25%。
#即使更大的shared_buffers有效，也会造成一些工作负 载， 但因为PostgreSQL同样依赖操作系统的高速缓冲区，
#将shared_buffers设置为超过 40% 的RAM不太可能比一个小点值工作得更好。
#为了能把对写大量新的或改变的数据的处 理分布在一个较长的时间段内，shared_buffers更大的设置通常要求对max_wal_size也 做相应增加。
#如果系统内存小于 1GB，一个较小的 RAM 百分数是合适的，这样可以为操作系统留下足 够的空间。	


#huge_pages = try			# on, off, or try
											# (change requires restart)
#控制是否为主共享内存区域请求巨型页。有效值是try(默认)、on以及off。
#如果huge_pages被设置为try，则服务器将尝试请求巨型页，但是如果失败会退回到默认的方式。
#如果为on，请求巨型页失败将使得服务器无法启动。
#如果为off，则不会请求巨型 页。
#当前，只有Linux和Windows上支持这个设置。在其他系统上这个参数被设置为try时，它会被忽略。
#巨型页面的使用会导致更小的页面表以及花费在内存管理上的 CPU 时间更少，从而提高 性能。


#huge_page_size = 0			# zero for system default
												# (change requires restart)
#temp_buffers = 8MB			# min 800kB

#为每个数据库会话设置用于临时缓冲区的最大内存.这些是仅用于访问临时表的会话本地 缓冲。 
#如果指定值时没有单位，则以块为单位，即BLCKSZ字节，通常为8kB。 默认为8 兆字节 (8MB)。(如果BLCKSZ不是8kB,则默认值按比例缩放。) 
#这个设置可以在独立的会 话内部被改变，但是只有在会话第一次使用临时表之前才能改变; 在会话中随后企图改变该值是无效的。
#一个会话将按照temp_buffers给出的限制根据需要分配临时缓冲区。
#如果在一个并不需要大量临时缓冲区的会话里设置一个大的数值，其开销只是一个缓冲区描述符，或者说temp_buffers每增加一则增加大概64字节。
#不过，如果一个缓冲区被实际使用，那么它就会额外消耗8192字节(或者BLCKSZ字节)。

#max_prepared_transactions = 0		# zero disables the feature
																	# (change requires restart)
																	

#设置可以同时处于“prepared”状态的事务的最大数目(见PREPARE TRANSACTION为两阶段提交准备当前事务)。
#把这个参数设置为零(这是默认设置)将禁用预备事务特性。这个参数只能在服务器启动 时设置。
#如果你不打算使用预备事务，可以把这个参数设置为零来防止意外创建预备事务。
#如果你正在使用预备事务，你将希望把max_prepared_transactions至少设置为max_connections一样大，因此每一个会话可以有一个预备事务#待处理。
#当运行一个后备服务器时，这个参数必须至少与主服务器上的一样大。否则，后备服务 器上将不会执行查询。																	
# Caution: it is not advisable to set max_prepared_transactions nonzero unless
# you actively intend to use prepared transactions.

#work_mem = 4MB				# min 64kB

#设置在写入临时磁盘文件之前查询操作(例如排序或哈希表)可使用的最大内存容量。 
#如 果指定值时没有单位，则以千字节为单位。默认值是4兆字节 (4MB)。 
#注意对于一个复杂查询，可能会并行运行好几个排序或者哈希操作;每个操作都会被允许使用这个参数指定的内存量，然后才会开始写数据到临时文件。#同样，几个正在运行的会话可能并发进行这样的操作。因此被使用的总内存可能是work_mem值的好几倍，在选择这个值时一定要记住这一点。
#ORDER BY、DISTINCT和归并连接都要用到排序操作。哈希连接、基于 哈希的聚集以及基于哈希的IN子查询处理中都要用到哈希表。

#hash_mem_multiplier = 1.0		# 1-1000.0 multiplier on hash table work_mem

#用于计算基于哈希的操作可以使用的最大内存量。最后的限制由work_mem乘以hash_mem_multiplier确定。
#默认值为1.0，这使得基于散列的操作与基于排序的操作具有相同的简单work_mem最大值。

#maintenance_work_mem = 64MB		# min 1MB

#指定在维护性操作(例如VACUUM、CREATE INDEX和ALTER TABLE ADD FOREIGN KEY) 中使用的 最大的内存量。
#如果指定值时没有单位，则以千字节为单位，其默认值是 64 兆字节(64MB)。
#因为在一个数据库会话中，一个时刻只有一个这样的操作可以被 执行，并且一个数据库安装通常不会有太多这样的操作并发执行，把这个数值设置得 #比work_mem大很多是安全的。更大的设置可以改进清理和恢复数据库转储的性能。
#注意当自动清理运行时，可能会分配最多达这个内存的autovacuum_max_workers倍，因此要小心不要把该默认值设置得太高。 
#通过独立地设置autovacuum_work_mem可能会对 控制这种情况 有所帮助。


#autovacuum_work_mem = -1		# min 1MB, or -1 to use maintenance_work_mem

#指定每个自动清理工作者进程能使用的最大内存量。如果指定值时没有单位，则以千字 节为单位。
#其默认值为 -1，表示转而使用 maintenance_work_mem的值。当运行在其他 上下文环境中时，这个设置对VACUUM的行为没有影响。

#logical_decoding_work_mem = 64MB	# min 64kB

#指定在将某些已解码的更改写入本地磁盘之前，逻辑解码要使用的最大内存量。这限制了逻辑流复制连接使用的内存量。它默认为64兆字节（64MB）。#由于每个复制连接只使用一个这样大小的缓冲区，而且一个安装通常不会同时有很多这样的连接（受max_wal_senders的限制），
#因此可以安全地将此值设置为远高于work_mem，从而减少写入磁盘的解码更改量。

#max_stack_depth = 2MB			# min 100kB

#指定服务器的执行堆栈的最大安全深度。这个参数的理想设置是由内核强制的实际栈尺寸限制(ulimit -s所设置的或者本地等价物)，减去大约一兆字#节的安全边缘。 需要 这个安全边缘是因为在服务器中并非所有例程都检查栈深度，只是在关键的可能递规的例程中才进行检查。
#如果指定值时没有单位，则以千字节为单位。默认设置是两兆字节 (2MB)，这个值相对比较小并且不可能导致崩溃。
#但是，这个值可能太小了，以至于无 法执行复杂的函数。只有超级用户可以修改这个设置。
#把max_stack_depth参数设置得高于实际的内核限制将意味着一个失控的递归函数可能会 导致一个独立的后端进程崩溃。 
#在PostgreSQL能够检测内核限制的平台上， 服务器将 不允许把这个参数设置为一个不安全的值。
#不过，并非所有平台都能提供该信息，所以 我们还是建议你在选择值时要小心。

#shared_memory_type = mmap		# the default is the first option
															# supported by the operating system:
															#   mmap
															#   sysv
															#   windows
															# (change requires restart)
#指定服务器应用于主共享内存区域的共享内存实现，包括 PostgreSQL 的共享缓冲 区和其他共享数据。 
#可能的值为 mmap (对使用 mmap 分配的匿名共享内 存)，
#sysv (通过 shmget 分配的系统V 共享内存)
#windows (Windows共享 内存)。 
#并非在所有平台上都支持全部值;第一个被支持的选项是该平台的默认选项。 
#sysv 选项不是任何平台的默认选项，通常不建议使用，因为它通常需要非默认的内核设 置来允许大量的地址分配。															
dynamic_shared_memory_type = posix	# the default is the first option
					# supported by the operating system:
					#   posix
					#   sysv
					#   windows
					#   mmap
					# (change requires restart)
					
#指定服务器应该使用的动态共享内存实现。可能的值是
#posix(用于使用 shm_open分 配的 POSIX 共享内存)、
#sysv (用于通过shmget分配的 System V 共享内存)、 
#windows(用于 Windows 共享内存)、 
#和mmap (使用存储在数据目录中的内存映射文件模拟共享内存)。

#并非所有平台上都支 持所有值，平台上第一个支持的选项就是其默认值。 
#在任何平台上mmap选项都不是默认 值，通常不鼓励使用它，因为操作系统会 反复地把修改过的页面写回到磁盘上，从而增 加了系统的I/O负载。
#不过当 pg_dynshmem目录被存储在一个 RAM 盘时或者没有其他共 享内存功能可用时， 它还是有用的。

#min_dynamic_shared_memory = 0MB	# (change requires restart)

#指定在服务器启动时应分配给并行查询使用的内存量。当此内存区域不足或并发查询耗尽时，新的并行查询会尝试使用#dynamic_shared_memory_type配置的方法从操作系统临时分配额外的共享内存，由于内存管理开销，这种方法可能会较慢。
#在启动时使用min_dynamic_shared_memory分配的内存受支持该设置的操作系统上huge_pages设置的影响，
#并且可能更容易从自动管理的操作系统上的大页面中受益。
#默认值为0（无）。此参数只能在服务器启动时设置。
```

#### 2.3.2 磁盘

```
# - Disk -

#temp_file_limit = -1			# limits per-process temp file space
													# in kilobytes, or -1 for no limit
													
#指定一个进程能用于临时文件(如排序和哈希临时文件，或者用于保持游标的存储文 件)的最大磁盘空间量。一个试图超过这个限制的事务将被取消。 #如果指定值时没有单 位，则以千字节为单位。-1(默认值)意味着没有限制。只有超级用户能够修改这个设 置。
#这个设置约束着一个给定PostgreSQL进程在任何瞬间所使用的所有临时文件的总空间。
#应该注意的是，与在查询执行中在幕后使用的临时文件相反，显式临时表所用的磁盘空间不被这个设置所限制。
```

#### 2.3.3 核心资源

```
# - Kernel Resources -

#max_files_per_process = 1000		# min 64
																# (change requires restart)

#设置每个服务器子进程允许同时打开的最大文件数目。默认是 1000 个文件。
#如果内 核强制一个安全的针对每个进程的限制，那么你不用操心这个设置。
#但是在 一些平台上 (特别是大多数 BSD 系统)，如果很多进程都尝试打开很多文件，
#内核将允许独立进程 打开比个系统真正可以支持的数目大得多得文件数。
#如果你发现自己看到了“Too many open files”这样的失败，可尝试减小这个设置。这个参数只能在服务器启动时设置。
```

#### 2.3.4 基于代价的清理延迟

```
# - Cost-Based Vacuum Delay -

#在VACUUM和ANALYZE命令的执行过程中，系统维持着一个内部计数器来跟踪各种被执行的I/O 操作的估算开销。当累计的代价达到一个限制(由#vacuum_cost_limit指定)，执行这些操 作的进程将按照vacuum_cost_delay所指定的休眠一小段时间。然后它将重置计数器并继续 执行。
#这个特性的出发点是允许管理员降低这些命令对并发的数据库活动产生的I/O影响。
#在很多情况下，VACUUM和ANALYZE等维护命令能否快速完成并不重要，
#而非常重要的是这些命令不会对系统执行其他数据库操作的能力产生显著的影响。基于代价的清理延迟提供了一种方式 让管理员能够保证这一点。
#对于手动发出的VACUUM命令，该特性默认被禁用。要启用它，只要把vacuum_cost_delay变 量设为一个非零值。


#vacuum_cost_delay = 0			# 0-100 milliseconds (0 disables)

#当超出开销限制时进程将要休眠的时间量。如果指定值时没有单位，则以毫秒为单位。 
#其默认值为0，这将禁用基于代价的清理延迟特性。正值将启用基于代价的清理。
#在使用基于代价的清理时，vacuum_cost_delay的合适值通常很小，也许是小于1毫秒。 
#虽然vacuum_cost_delay可以被设置为毫秒级别的值，但是在较老的平台上可能无法准确地测量这种延迟。 
#在这样的平台上，增加 VACUUM的节流资源消耗在1ms以上，需要改变 其他的清理开销参数。 
#尽管如此，你应该保持 vacuum_cost_delay 在平台能持续测量 的情况下尽可能小;大延迟没有帮助。
   
#vacuum_cost_page_hit = 1		# 0-10000 credits

#清理一个在共享缓存中找到的缓冲区的估计代价。它表示锁住缓冲池、查找共享哈希表 和扫描页内容的代价。默认值为1。

#vacuum_cost_page_miss = 2		# 0-10000 credits

#清理一个必须从磁盘上读取的缓冲区的代价。它表示锁住缓冲池、查找共享哈希表、从 磁盘读取需要的块以及扫描其内容的代价。默认值为10。

#vacuum_cost_page_dirty = 20		# 0-10000 credits

#当清理修改一个之前干净的块时需要花费的估计代价。它表示再次把脏块刷出到磁盘所 需要的额外I/O。默认值为20。

#vacuum_cost_limit = 200		# 1-10000 credits

#将导致清理进程休眠的累计代价。默认值为200。


#*有些操作会保持关键性的锁，这样可以尽快完成。基于代价的清理延迟在这类 操作期间不会发生。因此有可能代价会累计至大大超过指定的限制。
#为了防止 在这种情况下的无意义的长时间延迟，
#实际延迟的计算方式 是vacuum_cost_delay * accumulated_balance / vacuum_cost_limit，且最大 值是vacuum_cost_delay * 4。

```

#### 2.3.5 后台写入器

```
# - Background Writer -

#有一个独立的服务器进程，叫做后台写入器，它的功能就是发出写“脏”(新的或修改过 的)共享缓冲区的命令。
#它写出共享缓冲区，这样让处理用户查询的服务器进程很少或者永 不等待写动作的发生。不过，后台写入器确实会增加 I/O 的总负荷，
#因为虽然在每个检查点 间隔中一个重复弄脏的页面可#能只会写出一次，但在同一个间隔中后台写入器可能会把它写 出好几次。

#bgwriter_delay = 200ms			# 10-10000ms between rounds

#指定后台写入器活动轮次之间的延迟。在每个轮次中，写入器都会为一定数量的脏缓冲 区发出写操作(可以用下面的参数控制)。 
#然后它就休眠 bgwriter_delay的时长， 然后重复动作。当缓冲池中没有脏缓冲区时，不管 bgwriter_delay，它都会进入更长的休 眠。
#如果指定值时没有单位，则以毫秒为单位。默认值是 200 毫秒(200ms)。 
#注意在 许多系统上，休眠延迟的有效解析度是 10 毫秒;
#因此，为bgwriter_delay设置一个不是10的倍数的值与把它设置为下一个更高的 10 的倍数是一样的效果。
#这个选项只能 在服务器命令行上或者在postgresql.conf文件中设置。

#bgwriter_lru_maxpages = 100		# max buffers written/round, 0 disables

#在每个轮次中，不超过这么多个缓冲区将被后台写入器写出。
#把这个参数设置为零可禁 用后台写出(注意被一个独立、专用辅助进程管理的检查点不受影响)。默认值是 100 个缓冲区。
#这个参数只能在postgresql.conf文件中或在服务器命令行上设置。

#bgwriter_lru_multiplier = 2.0		# 0-10.0 multiplier on buffers scanned/round

#每一轮次要写的脏缓冲区的数目基于最近几个轮次中服务器进程需要的新缓冲区的数目。 
#最近所需的平均值乘以bgwriter_lru_multiplier可以估算下一轮次中将会需要的缓冲区数目。
#脏缓冲区将被写出直到有很多干净可重用的缓冲区(然而，每一轮次中写 出的缓冲区数不超过bgwriter_lru_maxpages)。 
#因此，设置为 1.0 表示一种“刚刚好 的”策略，这种策略会写出正好符合预测值的数目的缓冲区。 
#更大大的值可以为需求高 峰提供某种缓冲，而更小的值则需要服务进程来处理一些写出操作。默认值是 2.0。
#这个参数只能在postgresql.conf文件中或在服务器命令行上设置。


#bgwriter_flush_after = 512kB		# measured in pages, 0 disables

#只要后台写入的数据超过这个数量，尝试强制 OS 把这些写发送到底层存储上。
#这 样做将限制内核页缓存中脏数据的量，降低了在检查点末尾发出一个 fsync 时或 者 OS 在后台大批量写回数据时卡住的可能性。 
#那常常会导致大幅度压缩的事务延 迟，但是也有一些情况(特别是负载超过shared_buffers但小于 OS 页面高速缓存) 的性能会降低。
#这种设置可能会在某些平台上没有效果。 如果指定值时没有单位，则 以块为单位，即为BLCKSZ 字节，通常为8kB.
#合法的范围在0(禁用受控写回)和2MB之 间。Linux 上的默认值是512kB，其他平台上是0
#(如果BLCKSZ不是8kB，则默认值和最大 值会按比例缩放至这个值)。这个参数只能在postgresql.conf文件中或者服务器命令行 上设置。

#较小的bgwriter_lru_maxpages和bgwriter_lru_multiplier可以降低由后台写入器造成的额 外 I/O 开销。
#但更可能的是，服务器进程将必须自己发出写入操作，这会延迟交互式查询。
```

#### 2.3.6 异步行为

```
# - Asynchronous Behavior -

#backend_flush_after = 0		# measured in pages, 0 disables

#只要一个后端写入的数据量超过这个数量时，就会尝试强制 OS 把这些写发送到底层存 储。
#这样做将会限制内核页高速缓存中的脏数据数量，降低在检查点末尾发出fsync时或 者 OS 在后台大批写回数据时卡住的可能性。
#这常常会导致极大降低的事务延迟，但是 也有一些情况中(特别是负载超过shared_buffers但低于 OS 的页面高速缓存时)，性 能可能会下降。
#这个设置可能在某些平台上没有效果。 如果指定值时没有单位，则以块 为单位，即为 BLCKSZ 字节，通常为8kB。
#合法的范围位于0(禁用受控写回)和2MB之 间。默认是0(即没有强制写回)。(如果BLCKSZ不是8kB，最大值会按比例缩放到 它)。

#effective_io_concurrency = 1		# 1-1000; 0 disables prefetching

#设置PostgreSQL可以同时被执行的并发磁盘 I/O 操作的数量。调高这个值，可以增加任何单个PostgreSQL会话试图并行发起的I/O操作的数目。 #允许的范围是 1 到 1000， 或 0 表示禁用异步 I/O 请求。当前这个设置仅影响位图堆扫描。
#对于磁盘驱动器，这个设置的一个很好的出发点是组成一个被用于该数据库的RAID0条带或RAID1镜像的独立驱动器数量(RAID5校验驱动器不计入)。
#但是， 如果数据库经常忙于在并发会话中发出的多个查询，较低的值可能足以使磁盘阵列 繁忙。
#比保持磁盘繁忙所需的值更高的值只会造成额外的CPU开销。SSD以及其他基于内存的存储常常能处理很多并发请求，因此它们的最佳值可能是数百。
#异步I/O依赖于一个有效的posix_fadvise函数(一些操作系统可能没有)。如果不存在这个函数，将这个参数设置为除0之外的任何东西将导致错误。#在一些操作系统上 (如Solaris)虽然提供了这个函数，但它不会做任何事情。
#在支持的系统上默认值为 1，否则为 0。对于一个特定表空间中的表，可以通过设定该表空间的同名参数(见ALTER TABLESPACE)可以覆盖这个值。

#maintenance_io_concurrency = 10	# 1-1000; 0 disables prefetching

#类似于effective_io_concurrency，但用于代表许多客户端会话进行的维护工作。
#在支持的系统上，默认值为10，否则为0。对于特定表空间中的表，可以通过设置相同名称的表空间参数来覆盖该值（请参见ALTER tablespace）。

#max_worker_processes = 8		# (change requires restart)

#设置系统能够支持的后台进程的最大数量。这个参数只能在服务器启动时设置。默认值 为 8。
#在运行一个后备服务器时，你必须把这个参数设置为等于或者高于主控服务器上的值。 否则， 后备服务器上可能不会允许查询。
#更改这个值时考虑对max_parallel_workers、max_parallel_maintenance_workers、max_parallel_workers_per_gather进行调整。

#max_parallel_workers_per_gather = 2	# taken from max_parallel_workers

#设置单个Gather或者Gather Merge节点能够开始的工作者的最大数量。并行工作者会从max_worker_processes建立的进程池中取得，
#数量由max_parallel_workers限制。注 意所要求的工作者数量在运行时可能实际无法被满足。
#如果这种事情发生，该计划将会 以比预期更少的工作者运行，这可能会不太高效。默认值是2。
#把这个值设置为 0(默认 值)将会禁用并行查询执行。
#注意并行查询可能消耗比非并行查询更多的资源，因为每一个工作者进程时一个完全独立的进程,它对系统产生的影响大致和一个额外的用户会话相同。
#在为这个设置选择值时，以及配置其他控制资源利用的设置(例如work_mem)时，应该把这个因素考虑在 内。
#work_mem之类的资源限制会被独立地应用于每一个工作者，这意味着所有进程的总 资源利用可能会比单个进程时高得多。
#例如，一个使用 4 个工作者的并行查询使用的 CPU 时间、内存、I/O 带宽可能是不使用工作者时的 5 倍之多。
   
#max_parallel_maintenance_workers = 2	# taken from max_parallel_workers

#设置单一工具性命令能够启动的并行工作者的最大数目。当前，唯一一种支持使用并行工作者的工具性命令是CREATE INDEX，
#并且只有在构建B-树索引时才能并行。并行工作 者从由max_worker_processes创建的进程池中取出，数量由max_parallel_workers控 制。
#注意实际在运行时所请求数量的工作者可能不可用。如果发生这种情况，工具性操 作将使用比预期数量少的工作者运行。
#默认值为2。将这个值设置为0可以禁用工具性命令对并行工作者的使用。

#注意并行工具性命令不应该消耗比同等数量非并行操作更多的内存。这种策略与并行查询不同，并行查询的资源限制通常是应用在每个工作者进程上。
#并行工具性命令把资源限制maintenance_work_mem当作对整个工具性命令的限制，而不管其中用到了多少个并 行工作者进程。
#不过，并行工具性命令实际上可能仍会消耗更多的CPU资源和I/O带宽。

#max_parallel_workers = 8		# maximum number of max_worker_processes that
														# can be used in parallel operations
														
#设置系统为并行操作所支持的工作者的最大数量。默认值为8。
#在增加或者减小这个值 时，也要考虑对max_parallel_maintenance_workers以 及max_parallel_workers_per_gather进行调整。
#此外，要注意将这个值设置得大于max_worker_processes将不会产生效果，
#因为并行工作者进程都是从max_worker_processes所建立的工作者进程池中取出来的。

#parallel_leader_participation = on

#允许领导者进程执行查询，而不是等待工作进程。默认设置为on

#old_snapshot_threshold = -1		# 1min-60d; -1 disables; 0 is immediate
																# (change requires restart)
																
#设置可以使用查询快照的最小时间量，以规避使用快照时出现“snapshot too old” 错 误的风险。超过此阈值时间的数据将可以被清除。 这可以有#助于阻止长时间使用的快照 造成的快照膨胀。为了阻止由于本来对该快照可见的数据被清理导致的不正确结果，当 快照比这个阈值更旧并且该快照被#用来读取一个该快照建立以来被修改过的页面时，将 会产生一个错误。
#如果指定值时没有单位，则以分钟为单位。 值 -1 (默认值) 禁用此功能,实际上将快照 的时限设置为无穷大。 这个参数只能在服务器启动时设置。
#对于生产工作有用的值的范围可能从几个小时到几天不等。只允许小的值(例如0或 者1min)是因为它们有时可能对测试有用。 虽然允许高达60d的设#置，但是请注意很多 负载情况下，很短的时间帧里就可能发生极大的膨胀或者事务 ID 回卷。
#当这个特性被启用时，关系末尾的被清出的空间不能被释放给操作系统，因为那可能会 移除用于检测“snapshot too old”情况所需的信息。所有分#配给关系的空间还将与该 关系关联在一起便于重用，除非它们被显式地释放(例如，用VACUUM FULL)。

#这个设置不会尝试保证在任何特殊情况下都会生成错误。事实上，如果(例如)可以从 一个已经物化了一个结果集的游标中生成正确的结果，
#即便被引用表中的底层行已经被 清理掉也不会生成错误。某些表不能被过早地安全清除，并且因此将不受这个设置的影 响，例如系统目录。
#对于这些表，这个设置将不能降低膨胀，也不能降低在扫描时产 生“snapshot too old”错误的可能性。
```

### 2.4 预写式日志

#### 2.4.1 设置

```
#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# - Settings -

#wal_level = replica			# minimal, replica, or logical
					# (change requires restart)
#fsync = on				# flush data to disk for crash safety
					# (turning this off can cause
					# unrecoverable data corruption)
#synchronous_commit = on		# synchronization level;
					# off, local, remote_write, remote_apply, or on
#wal_sync_method = fsync		# the default is the first option
					# supported by the operating system:
					#   open_datasync
					#   fdatasync (default on Linux and FreeBSD)
					#   fsync
					#   fsync_writethrough
					#   open_sync
#full_page_writes = on			# recover from partial page writes
#wal_log_hints = off			# also do full page writes of non-critical updates
					# (change requires restart)
#wal_compression = off			# enable compression of full-page writes
#wal_init_zero = on			# zero-fill new WAL files
#wal_recycle = on			# recycle WAL files
#wal_buffers = -1			# min 32kB, -1 sets based on shared_buffers
					# (change requires restart)
#wal_writer_delay = 200ms		# 1-10000 milliseconds
#wal_writer_flush_after = 1MB		# measured in pages, 0 disables
#wal_skip_threshold = 2MB

#commit_delay = 0			# range 0-100000, in microseconds
#commit_siblings = 5			# range 1-1000
```

#### 2.4.2 检查点

```
# - Checkpoints -

#checkpoint_timeout = 5min		# range 30s-1d
#checkpoint_completion_target = 0.9	# checkpoint target duration, 0.0 - 1.0
#checkpoint_flush_after = 256kB		# measured in pages, 0 disables
#checkpoint_warning = 30s		# 0 disables
max_wal_size = 1GB
min_wal_size = 80MB
```

#### 2.4.3 归档

```
# - Archiving -

#archive_mode = off		# enables archiving; off, on, or always
											# (change requires restart)
#archive_command = ''		# command to use to archive a logfile segment
                        # placeholders: %p = path of file to archive
                        #               %f = file name only
                        # e.g. 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
#archive_timeout = 0		# force a logfile segment switch after this
												# number of seconds; 0 disables
```

#### 2.4.4 归档恢复

```
# - Archive Recovery -

# These are only used in recovery mode.

#restore_command = ''		# command to use to restore an archived logfile segment
                        # placeholders: %p = path of file to restore
                        #               %f = file name only
                        # e.g. 'cp /mnt/server/archivedir/%f %p'
#archive_cleanup_command = ''	# command to execute at every restartpoint
#recovery_end_command = ''	# command to execute at completion of recovery
```

#### 2.4.5 恢复目标

```
# - Recovery Target -

# Set these only when performing a targeted recovery.

#recovery_target = ''		# 'immediate' to end recovery as soon as a
                        # consistent state is reached
												# (change requires restart)
#recovery_target_name = ''	# the named restore point to which recovery will proceed
														# (change requires restart)
#recovery_target_time = ''	# the time stamp up to which recovery will proceed
														# (change requires restart)
#recovery_target_xid = ''	# the transaction ID up to which recovery will proceed
													# (change requires restart)
#recovery_target_lsn = ''	# the WAL LSN up to which recovery will proceed
													# (change requires restart)
#recovery_target_inclusive = on # Specifies whether to stop:
                                # just after the specified recovery target (on)
                                # just before the recovery target (off)
                                # (change requires restart)
#recovery_target_timeline = 'latest'	# 'current', 'latest', or timeline ID
																			# (change requires restart)
#recovery_target_action = 'pause'	# 'pause', 'promote', 'shutdown'
																	# (change requires restart)
```

### 2.5 流复制

#### 2.5.1 发送服务器

```
#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

# - Sending Servers -

# Set these on the primary and on any standby that will send replication data.

#max_wal_senders = 10		# max number of walsender processes
				# (change requires restart)
#max_replication_slots = 10	# max number of replication slots
				# (change requires restart)
#wal_keep_size = 0		# in megabytes; 0 disables
#max_slot_wal_keep_size = -1	# in megabytes; -1 disables
#wal_sender_timeout = 60s	# in milliseconds; 0 disables
#track_commit_timestamp = off	# collect timestamp of transaction commit
				# (change requires restart)
```

#### 2.5.2 主服务器

```
# - Primary Server -

# These settings are ignored on a standby server.

#synchronous_standby_names = ''	# standby servers that provide sync rep
				# method to choose sync standbys, number of sync standbys,
				# and comma-separated list of application_name
				# from standby(s); '*' = all
#vacuum_defer_cleanup_age = 0	# number of xacts by which cleanup is delayed
```

#### 2.5.3 备服务器

```
# - Standby Servers -

# These settings are ignored on a primary server.

#primary_conninfo = ''			# connection string to sending server
#primary_slot_name = ''			# replication slot on sending server
#promote_trigger_file = ''		# file name whose presence ends recovery
#hot_standby = on			# "off" disallows queries during recovery
					# (change requires restart)
#max_standby_archive_delay = 30s	# max delay before canceling queries
					# when reading WAL from archive;
					# -1 allows indefinite delay
#max_standby_streaming_delay = 30s	# max delay before canceling queries
					# when reading streaming WAL;
					# -1 allows indefinite delay
#wal_receiver_create_temp_slot = off	# create temp slot if primary_slot_name
					# is not set
#wal_receiver_status_interval = 10s	# send replies at least this often
					# 0 disables
#hot_standby_feedback = off		# send info from standby to prevent
					# query conflicts
#wal_receiver_timeout = 60s		# time that receiver waits for
					# communication from primary
					# in milliseconds; 0 disables
#wal_retrieve_retry_interval = 5s	# time to wait before retrying to
					# retrieve WAL after a failed attempt
#recovery_min_apply_delay = 0		# minimum delay for applying changes during recovery


```

#### 2.5.4 订阅者

```
# - Subscribers -

# These settings are ignored on a publisher.

#max_logical_replication_workers = 4	# taken from max_worker_processes
					# (change requires restart)
#max_sync_workers_per_subscription = 2	# taken from max_logical_replication_workers
```

### 2.6 查询规则

#### 2.6.1 规划器方法配置

```
#------------------------------------------------------------------------------
# QUERY TUNING
#------------------------------------------------------------------------------

# - Planner Method Configuration -

#enable_async_append = on
#enable_bitmapscan = on
#enable_gathermerge = on
#enable_hashagg = on
#enable_hashjoin = on
#enable_incremental_sort = on
#enable_indexscan = on
#enable_indexonlyscan = on
#enable_material = on
#enable_memoize = on
#enable_mergejoin = on
#enable_nestloop = on
#enable_parallel_append = on
#enable_parallel_hash = on
#enable_partition_pruning = on
#enable_partitionwise_join = off
#enable_partitionwise_aggregate = off
#enable_seqscan = on
#enable_sort = on
#enable_tidscan = on
```

#### 2.6.2 规划器代价常量

```
# - Planner Cost Constants -

#seq_page_cost = 1.0			# measured on an arbitrary scale
#random_page_cost = 4.0			# same scale as above
#cpu_tuple_cost = 0.01			# same scale as above
#cpu_index_tuple_cost = 0.005		# same scale as above
#cpu_operator_cost = 0.0025		# same scale as above
#parallel_setup_cost = 1000.0	# same scale as above
#parallel_tuple_cost = 0.1		# same scale as above
#min_parallel_table_scan_size = 8MB
#min_parallel_index_scan_size = 512kB
#effective_cache_size = 4GB

#jit_above_cost = 100000		# perform JIT compilation if available
					# and query more expensive than this;
					# -1 disables
#jit_inline_above_cost = 500000		# inline small functions if query is
					# more expensive than this; -1 disables
#jit_optimize_above_cost = 500000	# use expensive JIT optimizations if
					# query is more expensive than this;
					# -1 disables
```

#### 2.6.3 遗传查询优化

```
# - Genetic Query Optimizer -

#geqo = on
#geqo_threshold = 12
#geqo_effort = 5			# range 1-10
#geqo_pool_size = 0			# selects default based on effort
#geqo_generations = 0			# selects default based on effort
#geqo_selection_bias = 2.0		# range 1.5-2.0
#geqo_seed = 0.0			# range 0.0-1.0
```

#### 2.6.4 其他规划器选项

```
# - Other Planner Options -

#default_statistics_target = 100	# range 1-10000
#constraint_exclusion = partition	# on, off, or partition
#cursor_tuple_fraction = 0.1		# range 0.0-1.0
#from_collapse_limit = 8
#jit = on				# allow JIT compilation
#join_collapse_limit = 8		# 1 disables collapsing of explicit
					# JOIN clauses
#plan_cache_mode = auto			# auto, force_generic_plan or
					# force_custom_plan
```

### 2.7 报告和日志

#### 2.7.1 在哪里记日志

```
#------------------------------------------------------------------------------
# REPORTING AND LOGGING
#------------------------------------------------------------------------------

# - Where to Log -

#log_destination = 'stderr'		# Valid values are combinations of
					# stderr, csvlog, syslog, and eventlog,
					# depending on platform.  csvlog
					# requires logging_collector to be on.

# This is used when logging to stderr:
#logging_collector = off		# Enable capturing of stderr and csvlog
					# into log files. Required to be on for
					# csvlogs.
					# (change requires restart)

# These are only used if logging_collector is on:
#log_directory = 'log'			# directory where log files are written,
					# can be absolute or relative to PGDATA
#log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'	# log file name pattern,
					# can include strftime() escapes
#log_file_mode = 0600			# creation mode for log files,
					# begin with 0 to use octal notation
#log_rotation_age = 1d			# Automatic rotation of logfiles will
					# happen after that time.  0 disables.
#log_rotation_size = 10MB		# Automatic rotation of logfiles will
					# happen after that much log output.
					# 0 disables.
#log_truncate_on_rotation = off		# If on, an existing log file with the
					# same name as the new log file will be
					# truncated rather than appended to.
					# But such truncation only occurs on
					# time-driven rotation, not on restarts
					# or size-driven rotation.  Default is
					# off, meaning append to existing files
					# in all cases.

# These are relevant when logging to syslog:
#syslog_facility = 'LOCAL0'
#syslog_ident = 'postgres'
#syslog_sequence_numbers = on
#syslog_split_messages = on

# This is only relevant when logging to eventlog (Windows):
# (change requires restart)
#event_source = 'PostgreSQL'
```

#### 2.7.2 什么时候记日志

```
# - When to Log -

#log_min_messages = warning		# values in order of decreasing detail:
					#   debug5
					#   debug4
					#   debug3
					#   debug2
					#   debug1
					#   info
					#   notice
					#   warning
					#   error
					#   log
					#   fatal
					#   panic

#log_min_error_statement = error	# values in order of decreasing detail:
					#   debug5
					#   debug4
					#   debug3
					#   debug2
					#   debug1
					#   info
					#   notice
					#   warning
					#   error
					#   log
					#   fatal
					#   panic (effectively off)

#log_min_duration_statement = -1	# -1 is disabled, 0 logs all statements
					# and their durations, > 0 logs only
					# statements running at least this number
					# of milliseconds

#log_min_duration_sample = -1		# -1 is disabled, 0 logs a sample of statements
					# and their durations, > 0 logs only a sample of
					# statements running at least this number
					# of milliseconds;
					# sample fraction is determined by log_statement_sample_rate

#log_statement_sample_rate = 1.0	# fraction of logged statements exceeding
					# log_min_duration_sample to be logged;
					# 1.0 logs all such statements, 0.0 never logs


#log_transaction_sample_rate = 0.0	# fraction of transactions whose statements
					# are logged regardless of their duration; 1.0 logs all
					# statements from all transactions, 0.0 never logs
```

#### 2.7.3 记录什么到日志

```
# - What to Log -

#debug_print_parse = off
#debug_print_rewritten = off
#debug_print_plan = off
#debug_pretty_print = on
#log_autovacuum_min_duration = -1	# log autovacuum activity;
					# -1 disables, 0 logs all actions and
					# their durations, > 0 logs only
					# actions running at least this number
					# of milliseconds.
#log_checkpoints = off
#log_connections = off
#log_disconnections = off
#log_duration = off
#log_error_verbosity = default		# terse, default, or verbose messages
#log_hostname = off
#log_line_prefix = '%m [%p] '		# special values:
					#   %a = application name
					#   %u = user name
					#   %d = database name
					#   %r = remote host and port
					#   %h = remote host
					#   %b = backend type
					#   %p = process ID
					#   %P = process ID of parallel group leader
					#   %t = timestamp without milliseconds
					#   %m = timestamp with milliseconds
					#   %n = timestamp with milliseconds (as a Unix epoch)
					#   %Q = query ID (0 if none or not computed)
					#   %i = command tag
					#   %e = SQL state
					#   %c = session ID
					#   %l = session line number
					#   %s = session start timestamp
					#   %v = virtual transaction ID
					#   %x = transaction ID (0 if none)
					#   %q = stop here in non-session
					#        processes
					#   %% = '%'
					# e.g. '<%u%%%d> '
#log_lock_waits = off			# log lock waits >= deadlock_timeout
#log_recovery_conflict_waits = off	# log standby recovery conflict waits
					# >= deadlock_timeout
#log_parameter_max_length = -1		# when logging statements, limit logged
					# bind-parameter values to N bytes;
					# -1 means print in full, 0 disables
#log_parameter_max_length_on_error = 0	# when logging an error, limit logged
					# bind-parameter values to N bytes;
					# -1 means print in full, 0 disables
#log_statement = 'none'			# none, ddl, mod, all
#log_replication_commands = off
#log_temp_files = -1			# log temporary files equal or larger
					# than the specified size in kilobytes;
					# -1 disables, 0 logs all temp files
log_timezone = 'PRC'
```

#### 2.7.4 进程标题

```
#------------------------------------------------------------------------------
# PROCESS TITLE
#------------------------------------------------------------------------------

#cluster_name = ''			# added to process titles if nonempty
					# (change requires restart)
#update_process_title = on


```

### 2.8 统计信息

#### 2.8.1 查询索引统计收集器

```
#------------------------------------------------------------------------------
# STATISTICS
#------------------------------------------------------------------------------

# - Query and Index Statistics Collector -

#track_activities = on
#track_activity_query_size = 1024	# (change requires restart)
#track_counts = on
#track_io_timing = off
#track_wal_io_timing = off
#track_functions = none			# none, pl, all
#stats_temp_directory = 'pg_stat_tmp'
```

#### 2.8.2 统计信息监控

```
# - Monitoring -

#compute_query_id = auto
#log_statement_stats = off
#log_parser_stats = off
#log_planner_stats = off
#log_executor_stats = off
```

### 2.9 自动清理

```
#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------

#autovacuum = on			# Enable autovacuum subprocess?  'on'
					# requires track_counts to also be on.
#autovacuum_max_workers = 3		# max number of autovacuum subprocesses
					# (change requires restart)
#autovacuum_naptime = 1min		# time between autovacuum runs
#autovacuum_vacuum_threshold = 50	# min number of row updates before
					# vacuum
#autovacuum_vacuum_insert_threshold = 1000	# min number of row inserts
					# before vacuum; -1 disables insert
					# vacuums
#autovacuum_analyze_threshold = 50	# min number of row updates before
					# analyze
#autovacuum_vacuum_scale_factor = 0.2	# fraction of table size before vacuum
#autovacuum_vacuum_insert_scale_factor = 0.2	# fraction of inserts over table
					# size before insert vacuum
#autovacuum_analyze_scale_factor = 0.1	# fraction of table size before analyze
#autovacuum_freeze_max_age = 200000000	# maximum XID age before forced vacuum
					# (change requires restart)
#autovacuum_multixact_freeze_max_age = 400000000	# maximum multixact age
					# before forced vacuum
					# (change requires restart)
#autovacuum_vacuum_cost_delay = 2ms	# default vacuum cost delay for
					# autovacuum, in milliseconds;
					# -1 means use vacuum_cost_delay
#autovacuum_vacuum_cost_limit = -1	# default vacuum cost limit for
					# autovacuum, -1 means use
					# vacuum_cost_limit
```

### 2.10 客户端连接默认值

#### 2.10.1 语句行为

```
#------------------------------------------------------------------------------
# CLIENT CONNECTION DEFAULTS
#------------------------------------------------------------------------------

# - Statement Behavior -

#client_min_messages = notice		# values in order of decreasing detail:
					#   debug5
					#   debug4
					#   debug3
					#   debug2
					#   debug1
					#   log
					#   notice
					#   warning
					#   error
#search_path = '"$user", public'	# schema names
#row_security = on
#default_table_access_method = 'heap'
#default_tablespace = ''		# a tablespace name, '' uses the default
#default_toast_compression = 'pglz'	# 'pglz' or 'lz4'
#temp_tablespaces = ''			# a list of tablespace names, '' uses
					# only default tablespace
#check_function_bodies = on
#default_transaction_isolation = 'read committed'
#default_transaction_read_only = off
#default_transaction_deferrable = off
#session_replication_role = 'origin'
#statement_timeout = 0			# in milliseconds, 0 is disabled
#lock_timeout = 0			# in milliseconds, 0 is disabled
#idle_in_transaction_session_timeout = 0	# in milliseconds, 0 is disabled
#idle_session_timeout = 0		# in milliseconds, 0 is disabled
#vacuum_freeze_table_age = 150000000
#vacuum_freeze_min_age = 50000000
#vacuum_failsafe_age = 1600000000
#vacuum_multixact_freeze_table_age = 150000000
#vacuum_multixact_freeze_min_age = 5000000
#vacuum_multixact_failsafe_age = 1600000000
#bytea_output = 'hex'			# hex, escape
#xmlbinary = 'base64'
#xmloption = 'content'
#gin_pending_list_limit = 4MB
```

#### 2.10.2 区域和格式化

```
# - Locale and Formatting -

datestyle = 'iso, ymd'
#intervalstyle = 'postgres'
timezone = 'PRC'
#timezone_abbreviations = 'Default'     # Select the set of available time zone
					# abbreviations.  Currently, there are
					#   Default
					#   Australia (historical usage)
					#   India
					# You can create your own file in
					# share/timezonesets/.
#extra_float_digits = 1			# min -15, max 3; any value >0 actually
					# selects precise output mode
#client_encoding = sql_ascii		# actually, defaults to database
					# encoding

# These settings are initialized by initdb, but they can be changed.
lc_messages = 'en_US.UTF-8'			# locale for system error message
					# strings
lc_monetary = 'zh_CN.UTF-8'			# locale for monetary formatting
lc_numeric = 'zh_CN.UTF-8'			# locale for number formatting
lc_time = 'zh_CN.UTF-8'				# locale for time formatting

# default configuration for text search
default_text_search_config = 'pg_catalog.english'
```

#### 2.10.3 共享库预载入

```
# - Shared Library Preloading -

#local_preload_libraries = ''
#session_preload_libraries = ''
#shared_preload_libraries = ''	# (change requires restart)
#jit_provider = 'llvmjit'		# JIT library to use
```

#### 2.10.4 其他默认值

```
# - Other Defaults -

#dynamic_library_path = '$libdir'
#gin_fuzzy_search_limit = 0
```

### 2.11 锁管理

```
#------------------------------------------------------------------------------
# LOCK MANAGEMENT
#------------------------------------------------------------------------------

#deadlock_timeout = 1s
#max_locks_per_transaction = 64		# min 10
					# (change requires restart)
#max_pred_locks_per_transaction = 64	# min 10
					# (change requires restart)
#max_pred_locks_per_relation = -2	# negative values mean
					# (max_pred_locks_per_transaction
					#  / -max_pred_locks_per_relation) - 1
#max_pred_locks_per_page = 2            # min 0
```

### 2.12 版本和平台兼容性

#### 2.12.1 历史版本

```
#------------------------------------------------------------------------------
# VERSION AND PLATFORM COMPATIBILITY
#------------------------------------------------------------------------------

# - Previous PostgreSQL Versions -

#array_nulls = on
#backslash_quote = safe_encoding	# on, off, or safe_encoding
#escape_string_warning = on
#lo_compat_privileges = off
#quote_all_identifiers = off
#standard_conforming_strings = on
#synchronize_seqscans = on
```

#### 2.12.2 平台和客户端兼容性

```
# - Other Platforms and Clients -

#transform_null_equals = off
```

### 2.13 错误处理

```
#------------------------------------------------------------------------------
# ERROR HANDLING
#------------------------------------------------------------------------------

#exit_on_error = off			# terminate session on any error?
#restart_after_crash = on		# reinitialize after backend crash?
#data_sync_retry = off			# retry or panic on failure to fsync
					# data?
					# (change requires restart)
#recovery_init_sync_method = fsync	# fsync, syncfs (Linux 5.8+)
```

### 2.14 文件包含include

```
#------------------------------------------------------------------------------
# CONFIG FILE INCLUDES
#------------------------------------------------------------------------------

# These options allow settings to be loaded from files other than the
# default postgresql.conf.  Note that these are directives, not variable
# assignments, so they can usefully be given more than once.

#include_dir = '...'			# include files ending in '.conf' from
													# a directory, e.g., 'conf.d'
#include_if_exists = '...'		# include file only if it exists
#include = '...'							# include file
```

### 2.15 自定义选项

```
#------------------------------------------------------------------------------
# CUSTOMIZED OPTIONS
#------------------------------------------------------------------------------

# Add settings for extensions here
```

### 2.16 预置选项

```
下列“参数”是只读的，它们是在编译或安装PostgreSQL时决定的。同样，它们被排除 在postgresql.conf文件例子之外。这些选项报告特定应用可能感兴趣的多种PostgreSQL行 为，特别是管理前端相关的行为。
block_size (integer)
报告一个磁盘块的大小。它由编译服务器时BLCKSZ的值确定。默认值是 8192 字节。有 些配置变量的含义(例如shared_buffers)会被block_size影响。详见第 19.4 节。
data_checksums (boolean) 报告对这个集簇是否启用了数据校验码。详见data checksums。
data_directory_mode (integer)
在Unix系统上，这个参数报告启动时data_directory所定义的数据目录的权限(在 Microsoft Windows上这个参数将总是显示0700)。更多信息请参考group access。
debug_assertions (boolean)
报告编译PostgreSQL时是否启用了断言。 如果PostgreSQL被编译时定义了宏 USE_ASSERT_CHECKING is defined when PostgreSQL(例如通过 configure选项 --enable-cassert定义)，那么会报告已启用。默认情况下 PostgreSQL编译时没有用断 言。
integer_datetimes (boolean)
报告PostgreSQL是否在编译时打开了 64 位整数日期和时间。从PostgreSQL 10起，这个 值总是on。
lc_collate (string)
报告文本数据排序使用的区域。详见第 23.1 节。该值是在数据库被创建时确定的。
          lc_ctype (string)
报告决定字符分类的区域。详见第 23.1 节。该值是在数据库被创建时决定的。通常它 和lc_collate一样，但是可以为特殊应用设置成不同的值。
max_function_args (integer)
报告函数参数的最大数量。它由编译服务器时的FUNC_MAX_ARGS值决定的。默认值是 100 个参数。
max_identifier_length (integer)
报告标识符的最大长度。它由编译服务器时的NAMEDATALEN值减一决定。NAMEDATALEN的 默认值是 64 ;因此max_identifier_length的默认值是 63，但是在使用多字节编码时 可以少于 63 个字符。
max_index_keys (integer)
报告索引键的最大数目。它由编译服务器时的INDEX_MAX_KEYS值决定。默认值是 32 个 键。
segment_size (integer)
报告一个文件段中可以存储的块(页)的数量。由编译服务器时的RELSEG_SIZE值决定。 一个段文件的最大尺寸(以字节计)等于segment_size乘以block_size，默认是 1GB。
server_encoding (string)
报告数据库的编码(字符集)。这是在数据库被创建时决定的。通常，客户端只需要关 心client_encoding的值。
server_version (string) 报告服务器版本数值。它是由编译服务器时的PG_VERSION值决定的。
server_version_num (integer) 报告服务器版本数值的整数值。它是由编译服务器时的PG_VERSION_NUM值决定的。
ssl_library (string)
报告此PostgreSQL服务器已构建的 SSL 库的名称(即使此实例当前未配置或使用 SSL)， 例如OpenSSL，或一个空字符串(如果没有)。
wal_block_size (integer)
报告一个 WAL 磁盘块的尺寸。由编译服务器时的XLOG_BLCKSZ值决定。默认是 8192 字 节。
wal_segment_size (integer)
报告 WAL 段文件的大小。默认是 16MB。详见第 29.4 节。
```

### 2.17 开发者选项

```
下面的参数目的是用在PostgreSQL源代码上， 并且在某些情况下可以帮助恢复严重损坏了 的数据库。在一个生产数据库中没有理由使用它们。同样，它们被从例子postgresql.conf文件中排除。请注意许多这些参数要求特殊的源代码编译标志才能工 作。
allow_system_table_mods (boolean) 允许对系统表结构的修改。它可以被initdb使用。这个参数只能在服务器启动时设置。
ignore_system_indexes (boolean)
读取系统表时忽略系统索引(但是修改系统表时依然同时更新索引)。这在从被破坏的 系统索引中恢复数据的时有用。这个参数在会话开始之后不能被更改。
post_auth_delay (integer)
执行身份验证过程后启动新服务器进程时延迟的时间量。这是为了给开发者们一个机会 在一个服务器进程上附加一个调试器。 如果指定值时没有单位，则以秒为单位。0值(默 认值)禁用延迟。这个参数在会话开始之后不能被更改。
pre_auth_delay (integer)
在新服务器进程分叉后，在进行身份验证过程之前，延迟的时间量。这是为了给开发者 们一个机会在一个服务器进程上附加一个调试器来跟踪认证过程中的不当行为。 如果指 定值时没有单位，则以秒为单位。0值(默认值)禁用延迟。这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。
trace_notify (boolean)
为LISTEN和NOTIFY命令生成大量调试输出。client_min_messages和log_min_messages必 须是DEBUG1或者更低才能把这种输出分别发送到客户端或者服务器日志。
trace_recovery_messages (enum)
启用记录与恢复有关的调试输出，否则无法记录。这个参数允许用户覆 盖log_min_messages的正常设置，但只用于指定的消息。这个参数的目的是用来调试热 后备。有效值包括DEBUG5、DEBUG4、DEBUG3、DEBUG2、DEBUG1和LOG。默认值LOG完全不 会影响日志决定。其他值会记录相关级别或更高级别的与恢复相关的调试消息，就好像 它们具有LOG优先级一样;对于log_min_messages的通用设置，这会无条件的将消息发送 给服务器日志。这个参数只能在postgresql.conf文件中或在服务器命令行上设置。
trace_sort (boolean)
如果打开，发出在排序操作中的资源使用的相关信息。只有在编译PostgreSQL时定义 了TRACE_SORT宏， 这个参数才可用(不过，当前在默认情况下就定义了TRACE_SORT)。
trace_locks (boolean)
如果开启，发出锁使用情况的信息。被转储信息中包括锁操作的类型、锁的类型和 被锁 或被解锁对象的唯一标识符。同样包括的还有已经授予这个对象的锁类型的位掩码和 等 待这个对象的锁类型的位掩码。对每一种锁类型，已授权锁和等待锁的计数也会被一起 转储。一个日志文件输出的例子如下:
LOG: LockAcquire: new: lock(0xb7acd844) id(24688,24696,0,0,0,1) grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0 wait(0) type(AccessShareLock)
LOG: GrantLock: lock(0xb7acd844) id(24688,24696,0,0,0,1) grantMask(2) req(1,0,0,0,0,0,0)=1 grant(1,0,0,0,0,0,0)=1 wait(0) type(AccessShareLock)
LOG: UnGrantLock: updated: lock(0xb7acd844) id(24688,24696,0,0,0,1) grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0 wait(0) type(AccessShareLock)
LOG: CleanUpLock: deleting: lock(0xb7acd844) id(24688,24696,0,0,0,1) grantMask(0) req(0,0,0,0,0,0,0)=0 grant(0,0,0,0,0,0,0)=0 wait(0) type(INVALID)
被转储结构的详细信息可以在src/include/storage/lock.h中找到。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 trace_lwlocks (boolean)
如果开启，发出轻量级锁的使用信息。轻量级锁主要是为了提供对共享内存数据结构的 互斥访问。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 trace_userlocks (boolean)
如果开启，发出关于用户锁使用的信息。与trace_locks的输出一样，但只用于咨询锁。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 trace_lock_oidmin (integer)
如果设置，不会跟踪小于这个 OID 的锁(用于避免在系统表上的输出)。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 trace_lock_table (integer)
无条件地跟踪此表(OID)上的锁。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 debug_deadlocks (boolean)
如果设置，当死锁超时发生时，转储所有当前锁的信息。
只有在编译PostgreSQL时定义了LOCK_DEBUG宏， 这个参数才可用。 log_btree_build_stats (boolean)
如果设置，会记录 B 树操作上的系统资源使用情况统计(内存和 CPU)。
只有在编译PostgreSQL时定义了BTREE_BUILD_STATS宏， 这个参数才可用。 wal_consistency_checking (string)
这个参数被设计用来检查WAL重做例程中的缺陷。当这个参数被启用时，被修改的任何缓 冲区的全页映像及其WAL记录都被加入到记录中。如果该记录后来被重放，系统将首先应 用每个记录然后测试该记录修改的缓冲区是否符合存储的映像。在某些情况下(例如提 示位)，小的变动是可以接受的，并且会被忽略。任何预期之外的差别都将导致致命错 误，最后中止恢复。
这个设置的默认值是空字符串，它将禁用这一特性。它可以被设置为all以检查所有记 录，或者被设置为一个逗号分隔的资源管理器列表用以检查那些资源管理器产生的记 录。当前，支持的资源管理器 是heap、heap2、btree、hash、gin、gist、sequence、spgist、brin以及generic。只 有超级用户可以更改这一设置。
wal_debug (boolean)
如果被打开，WAL 相关的调试输出将被发出。只有在编译PostgreSQL时定义 了WAL_DEBUG宏的情况下，这个参数才可用。
ignore_checksum_failure (boolean) 只有当data checksums被启用时才有效。
在读取过程中检测到一次校验码失败通常会导致PostgreSQL报告一个错误。设 置ignore_checksum_failure为打开会导致系统忽略失败(但是仍然报告一个警告)，并 且继续执行。这种行为可能导致崩溃、传播或隐藏损坏或者其他严重的问题。但是，它 允许你绕过错误并且在块头部仍然健全的情况下从表中检索未损坏的元组。如果头部被 损坏，即便这个选项被启用系统也将报告一个错误。默认设置是off，并且只能被超级用 户改变。
zero_damaged_pages (boolean)
检测到一个损坏的页面头部通常会导致PostgreSQL报告一个错误，并且中止当前事务。 把zero_damaged_pages设置为打开会让系统报告一个警告、把损坏的页面填充零，然后 继续处理。这种行为会毁掉数据，即被损坏页面上的所有行。但是它允许你绕开错误并 且从可能存在表中的任何未损坏页面中检索行。如果由于一次硬件或软件错误而发生毁 坏，这种方法可用于恢复数据。通常你不应该把它设置为打开，除非你已经彻底放弃从 表的损坏页面中恢复数据。被填充零的页面不会被强制到磁盘上，因此我们推荐在再次 关闭这个参数之前先重建表或索引。默认的设置是off，并且只有超级用户可以改变它。
jit_debugging_support (boolean)
如果LLVM有所需要的功能，用GDB注册所生成的函数。这会让调试更加容易。默认设置 是off。这个参数只能在服务器启动时设置。
jit_dump_bitcode (boolean)
把生成的LLVM IR写出到文件系统，写到data_directory中。只有在做JIT内部实现工作 时，这个参数才能派上用场。默认设置是off。这个参数只能由超级用户修改。
jit_expressions (boolean)
当JIT编译被激活时(见第 31.2 节)，确定表达式是否用JIT编译。默认值是on。
jit_profiling_support (boolean)
如果LLVM有所需的功能，发出需要的数据以允许perf对JIT生成的函数画像。这会写出文 件到$HOME/.debug/jit/中，如果需要，由用户负责对其执行清除。默认设置是off。这 个参数只能在服务器启动时设置。
jit_tuple_deforming (boolean)
当JIT编译被激活时(见第 31.2 节)，确定元组拆解是否被JIT编译。默认值是on。
```

### 2.18 短选项

```
为了方便起见，系统中还为一些参数提供了单字母的命令行选项开关。其中一些选项是由于历史原因而存在，并且它们作为一个单字母选项存在并不表示它们 会被大量使用
```

