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
#wal_level决定多少信息写入到 WAL 中。默认值是replica，它会写入足够的数据以支持 WAL归档和复制，包括在后备服务器上运行只读查询。#minimal会去掉除从崩溃或者立即 关机中进行恢复所需的信息之外的所有记录。
#最后，logical会增加支持逻辑解码所需的 信息。每个层次包括所有更低层次记录的信息。这个参数只能在服务器启动时设置。
#在minimal级别中，某些批量操作的 WAL 日志可以被安全地跳过，这可以使那些操作更快。这种优化可以应用的操作包括:
#CREATE TABLE AS
#CREATE INDEX
#CLUSTER COPY到在同一个事务中被创建或截断的表中
#但最少的 WAL 不会包括足够的信息来从基础备份和 WAL 日志中重建数据，
#因此，要启 用 WAL 归档(archive_mode)和流复制，必须使用replica或更高级别。
#在logical层，与replica相同的信息会被记录，外加上 允许从 WAL 抽取逻辑修改集所需 的信息。
#使用级别 logical将增加 WAL 容量，特别是如果为了REPLICA IDENTITY FULL配 置了很多表并且执行了很多UPDATE和DELETE 语句时。
#在 9.6 之前的版本中，这个参数也允许值archive和hot_standby。现在仍然接受这些 值，但是它们会被映射到replica。													
#fsync = on				# flush data to disk for crash safety
									# (turning this off can cause
									# unrecoverable data corruption)
#如果打开这个参数，PostgreSQL服务器将尝试确保更新被物理地写入到磁盘，做法是发出fsync()系统调用或者使用多种等价的方法(见#wal_sync_method)。这保证了数据库 集簇在一次操作系统或者硬件崩溃后能恢复到一个一致的状态。
#虽然关闭fsync常常可以得到性能上的收益，但当发生断电或系统崩溃时可能造成不可恢 复的数据损坏。因此，只有在能很容易地从外部数据中重建整#个数据库时才建议关 闭fsync。
#能安全关闭fsync的环境的例子包括从一个备份文件中初始加载一个新数据库集簇、使用一个数据库集簇来在数据库被删掉并重建之后处理一批数据，
#或者一个被经常重建并却不用于失效备援的只读数据库克隆。单独的高质量硬件不足以成为关闭fsync的理由。
#当把fsync从关闭改成打开时，为了可靠的恢复，需要强制在内核中的所有被修改的缓冲 区进入持久化存储。
#这可以在多个时机来完成:在集簇被关闭时或在fsync因为运 行initdb --sync-only而打开时、运行sync时、卸载文件系统时或者重启服务器时。
#在很多情况下，为不重要的事务关闭synchronous_commit可以提供很多关闭fsync的潜在性能收益，并不会有的同时， 
#关闭fsync可以提供很多潜在的性能优势，而不会有伴随 着的数据损坏风险。
#fsync只能在postgresql.conf文件中或在服务器命令行上设置。如果你关闭这个参数， 请也考虑关闭full_page_writes。

#synchronous_commit = on		# synchronization level;
														# off, local, remote_write, remote_apply, or on
#指定在命令返回“success”指示给客户端之前，一个事务是否需要等待WAL记录被写入磁盘。合法的值是on、remote_apply、remote_write、#local和off。默认的并且安全的设置是on。
#当设置为off时，在向客户端报告成功和真正保证事务不会被服务器崩溃威 胁之间会有延迟(最大的延迟是wal_writer_delay的三倍)。
#不同于fsync，将这个参数设置为off不会产生数据库不一致性的风险:一个操作系统或数据库崩溃可能会造成一些最近据说已提交的事务丢失，
#但数据库状态是一致的，就像这些事务已经被干净地中 止。
#因此，当性能比完全确保事务的持久性更重要时，关闭synchronous_commit可以作 为一个有效的代替手段。

#如果synchronous_standby_names为非空，这个参数也控制事务提交是否将等待它们的 WAL 记录被复制到后备服务器上。
#当这个参数被设置为on时，直到来自当前同步的后备服务器的回复指示它们已经收到了事务的提交记录并将其刷入磁盘，主服务器上的事务才会提交。#这保证事务将不会被丢失，除非主服务器和所有同步后备都遭受到了数 据库存储损坏的问题。
#当被设置为remote_apply时，提交将会等待，直到来自当前的同步后备的回复指示它们已经收到了该事务的提交记录并且已经应用了该事务，
#这样该事 务才变得对后备上的查询可见。

#当这个参数被设置为remote_write时，提交将等待，直 到来自当前的同步后备的回复指示它们已经收到了该事务的提交记录并且已经把该记录写出
#到它们的操作系统，这种设置足以保证数据在后备服务器的PostgreSQL实例崩溃时得以保存，但是不能保证后备服务器遭受操作系统级别崩溃时数据
#能被保持，因为数据 不一定必须要在后备机上达到稳定存储。最后，设置local会导致提交等待本地刷写到磁 盘而不是复制完成。在使用同步复制时#这通常不是我们想要的效果，但是为了完整性， 还是提供了这样一个选项。

#如果synchronous_standby_names为空,设置on、remote_apply、remote_write和local都提供了同样的同步级别
#即事务提交只等待本地刷写磁盘。

#这个参数可以随时被修改;任何一个事务的行为由其提交时生效的设置决定。因此，可以同步提交一些事务，同时异步提交其他事务。
#例如，当默认是相反时，实现一个单一 多语句事务的异步提交，在事务中发出SET LOCAL synchronous_commit TO OFF。

#wal_sync_method = fsync		# the default is the first option
														# supported by the operating system:
                            #   open_datasync
                            #   fdatasync (default on Linux and FreeBSD)
                            #   fsync
                            #   fsync_writethrough
                            #   open_sync
#用来向强制 WAL 更新到磁盘的方法。如果fsync是关闭的，那么这个设置就不相关，因为WAL文件更新将根本不会被强制。可能的值是:
#• open_datasync(用open()选项O_DSYNC写 WAL 文件)
#• fdatasync(在每次提交时调用fdatasync())
#• fsync(在每次提交时调用fsync())
#• fsync_writethrough(在每次提交时调用fsync()，强制任何磁盘写高速缓存的直通 写)
#• open_sync(用open()选项O_SYNC写 WAL 文件)
#open_* 选项也可以使用O_DIRECT(如果可用)。不是在所有平台上都能使用所有这些选 择。默认值是列表中第一个被平台支持的那个，
#不过fdatasync是 Linux 中的默认值。默认值不一定是最理想的;有可能需要修改这个设置或系统配置的其他方面来创建一个崩溃-安全的配置，
#或达到最佳性能。这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。


#full_page_writes = on			# recover from partial page writes

#当这个参数为打开时，PostgreSQL服务器在一个检查点之后的页面的第一次修改期间将 每个页面的全部内容写到 WAL 中。
#这么做是因为在操作系统崩溃期间正在处理的一次页写入可能只有部分完成，从而导致在一个磁盘页面中混合有新旧数据。
#在崩溃后的恢复 期间，通常存储在 WAL 中的行级改变数据不足以完全恢复这样一个页面。
#存储完整的 页面映像可以保证页面被正确存储，但代价是增加了必须被写入 WAL 的数据量(因为 WAL 重放总是从一个检查点开始，
#所以在检查点后每个页面的第一次改变时这样做就够 了。因此，一种减小全页面写开销的方法是增加检查点间隔参数值)。
#把这个参数关闭会加快正常操作，但是在系统失败后可能导致不可恢复的数据损坏，或者静默的数据损坏。
#其风险类似于关闭fsync， 但是风险较小。并且只有在可关 闭fsync的情况下才应该关闭它。
#关闭这个选项并不影响用于时间点恢复(PITR)的 WAL 归档使用。
#这个参数只能在postgresql.conf文件中或在服务器命令行上设置。默认值是on。

#wal_log_hints = off			# also do full page writes of non-critical updates
													# (change requires restart)
#当这个参数为on时，PostgreSQL服务器一个检查点之后页面被第一次修改期间把该磁盘 页面的整个内容都写入 WAL，
#即使对所谓的提示位做非关键修改也会这样做。
#如果启用了数据校验和，提示位更新总是会被 WAL 记录并且这个设置会被忽略。
#你可以使用这个设置测试如果你的数据库启用了数据校验和，会有多少额外的 WAL 记录发生。
#这个参数只能在服务器启动时设置。默认值是off。	

#wal_compression = off			# enable compression of full-page writes

#当这个参数为on时，如果full_page_writes 为打开或者处于基础备份期间，PostgreSQL 服务器 会压缩写入到 WAL 中的完整页面镜像。
#压缩页面镜像将在 WAL 重放时 被解压。 默认值为off。只有超级用户可以更改这个设置。
#打开这个参数可以减小 WAL 所占的空间且无需承受不可恢复的数据损坏风险，
#但是代 价是需要额外的 CPU 开销以便在 WAL 记录期间进行压缩以及在 WAL 重放时解压。

#wal_init_zero = on			# zero-fill new WAL files 
#如果设置为on(默认值)，这个选项会导致新的WAL文件被0填满。 在某些文件系统上，这 可以确保在需要写入WAL记录之前分配空间。 
#不过，Copy-On-Write (COW)文件系统可能 不会受益于这种技术，所以给出这个选项可以跳过不必要的工作。 
#如果设置为off，则 在创建文件时只写入最后一个字节，使其具有预期的大小。

#wal_recycle = on			# recycle WAL files

#如果设置为 on (默认值), 这个选项会导致 WAL文件通过重命名它们来回收, 从而避免 创建新文件的需要。 
#在COW 文件系统, 创建新文件也许会更快，因此提供这个选项来禁 用这种行为。

#wal_buffers = -1			# min 32kB, -1 sets based on shared_buffers
											# (change requires restart)
#用于还未写入磁盘的 WAL 数据的共享内存量。默认值 -1 选择等于shared_buffers的 1/32 的尺寸(大约3%)，
#但是不小于64kB也不大于 WAL 段的尺寸(通常为)。如果自动的选择太大或太小可以手工设置该值，但是任何小于32kB的正值都将被当作32kB。
#如 果指定值时没有单位，则以WAL块作为单位，即为 XLOG_BLCKSZ 字节，通常为8kB。这个 参数只能在服务器启动时设置。

#在每次事务提交时，WAL 缓冲区的内容被写出到磁盘，因此极大的值不可能提供显著的 收益。
#不过，把这个值设置为几个兆字节可以在一个繁忙的服务器(其中很多客户端会 在同一时间提交)上提高写性能。
#由默认设置 -1 选择的自动调节将在大部分情况下得 到合理的结果。		

#wal_writer_delay = 200ms		# 1-10000 milliseconds

#指定 WAL 写入器刷写 WAL 的频繁程度，以时间为单 位。 
#在刷写WAL之后，写入器将根据wal_writer_delay所给出的时间长度进行睡眠，除 非被一个异步提交的事务提前唤醒。 
#如果最近的刷写发生在 wal_writer_delay 之 前，并且小于 wal_writer_flush_after WAL的值产生之后，那么WAL只会被写入操作 系统，
#而不会被刷写到磁盘。 如果指定值时没有单位，则以毫秒作为单位。 默认值 是 200 毫秒(200ms)。
#注意在很多系统上，有效的睡眠延迟粒度是 10 毫秒， 把wal_writer_delay设置为一个不是 10 的倍数的值，
#其效果和把它设置为大于该值的 下一个 10 的倍数产生的效果相同。
#这个参数只能在postgresql.conf文件中或者服务器 命令行上设置。

#wal_writer_flush_after = 1MB		# measured in pages, 0 disables

#指定 WAL 写入器刷写 WAL 的频繁程度，以卷为单位。 
#如果最近的刷写发生在 wal_writer_delay 之前，并且小于 wal_writer_flush_after WAL的值产生之后，
#那么WAL只会被写入操作系统，而不会被刷写到磁盘。 
#如果wal_writer_flush_after被设置 为0，则WAL数据总是会被立即刷写。 
#如果指定值时没有单位，则以WAL块作为单位，即 为XLOG_BLCKSZ字节，通常为8kB。 默认是1MB。
#这个参数只能在postgresql.conf文件中 或者服务器命令行上设置。

#wal_skip_threshold = 2MB

#When wal_level is minimal and a transaction commits after creating or rewriting a permanent relation, this #setting determines how to persist the new data. If the data is smaller than this setting, write it to the WAL #log; otherwise, use an fsync of affected files. Depending on the properties of your storage, raising or #lowering this value might help if such commits are slowing concurrent transactions.  
#如果指定此值时没有单位，则将其视为千字节。默认值为2兆字节（2MB）。

#commit_delay = 0			# range 0-100000, in microseconds

#在一次 WAL 刷写被发起之前，commit_delay增加一个时间延迟。 如果系统负载足够高， 使得在一个给定间隔内有额外的事务准备好提交，
#那么通过允许更多事务通过一个单次 WAL 刷写来提交能够提高组提交的吞吐量。 但是，它也把每次 WAL 刷写的潜伏期增加到 了最多#commit_delay。 因为如果没有其他事务准备好提交，就会浪费一次延迟，只有在 当一次刷写将要被发起时有至少commit_siblings个其他活动事#务时，才会执行一次延 迟。 另外，如果fsync被禁用，则将不会执行任何延迟。 如果指定值时没有单位，则 以微秒作为单位。 默认的#commit_delay是零(无延迟)。只有超级用户才能修改这个设 置。
#在PostgreSQL的 9.3 发布之前，commit_delay的行为不同并且效果更差:它只影响提 交，而不是所有 WAL 刷写，并且即使在 WAL 刷写马上就#要完成时也会等待一整个配置 的延迟。从PostgreSQL 9.3 中开始，第一个准备好刷写的进程会等待配置的间隔，而后 续的进程只等到领先者完成#刷写操作。

#commit_siblings = 5			# range 1-1000

#在执行commit_delay延迟时，要求的并发活动事务的最小数目。大一些的值会导致在延 迟间隔期间更可能有至少另外一个事务准备好提交。
#默认值是五个事务。
```

#### 2.4.2 检查点

```
# - Checkpoints -

#checkpoint_timeout = 5min		# range 30s-1d

#自动 WAL 检查点之间的最长时间。如果指定值时没有单位，则以秒为单位。 合理的范 围在 30 秒到 1 天之间。默认是 5 分钟(5min)。
#增加这个参数的值会增加崩溃恢复 所需的时间。这个参数只能在postgresql.conf文件中或在服务器命令行上设置。

#checkpoint_completion_target = 0.9	# checkpoint target duration, 0.0 - 1.0

#指定检查点完成的目标，作为检查点之间总时间的一部分。默认是 0.9。 这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。

#checkpoint_flush_after = 256kB		# measured in pages, 0 disables

#当执行检查点时写入的数据量超过此数量时，就尝试强制 OS 把这些 写发送到底层存储。 这样做将会限制内核页面高速缓存中的脏数据数量，
#降低 在检查点末尾发出fsync或者 OS 在后台大批量写回数据时被卡住的可 能性。 
#那常常会导致大幅度压缩的事务延迟，但是也有一些情况(特别是负载超过shared_buffers但小于OS页面高速缓存）的性能降低。
#这种设置可能会在某些平台上没有效果。
如果指定值时没有单位，则以块为单位，即为BLCKSZ 字节，通常为8kB。
#合法的范围在0（禁用强制协会）和2M之间。
#Linux 上的默认值是256kB，其他平台上是0(如果BLCKSZ不是8kB，则默认值和最大 值会按比例缩放到它)。
#这个参数只能在postgresql.conf文件中或者服务器命令行上设 置。


#checkpoint_warning = 30s		# 0 disables

#如果由于填充WAL段文件导致的检查点之间的间隔低于这个参数表示的时间量，那么就向 服务器日志写一个消息(它建议增加max_wal_size的值)。 #如果指定值时没有单位，则 以秒为单位。默认值是 30 秒(30s)。零则关闭警告。如果checkpoint_timeout低 于checkpoint_warning，则不#会有警告产生。这个参数只能在postgresql.conf文件中或 在服务器命令行上设置。

max_wal_size = 1GB

#在自动 WAL 检查点之间允许 WAL 增长到的最大尺寸。这是一个软限制， 在特殊的情况 下 WAL 尺寸可能会超过max_wal_size， 例如在重度负
#荷下、archive_command失败或者 高的 wal_keep_segments设置。如果指定值时没有单位，则以兆字节为单位。默认为 1 GB。增加这个参数 
#可能导致崩溃恢复所需的时间。这个参数只能在postgresql.conf 或 者服务器命令行中设置。

min_wal_size = 80MB

#只要 WAL 磁盘用量保持在这个设置之下，在检查点时旧的 WAL 文件总是 被回收以便未 来使用，而不是直接被删除。这可以被用来确保有足够的 #WAL 空间被保留来应付 WAL 使 用的高峰，例如运行大型的批处理任务。 如果指定值时没有单位，则以兆字节为单位。 默认是 80 MB。这个参数
#只能在postgresql.conf 或者服务器命令行中设置。
```

#### 2.4.3 归档

```
# - Archiving -

#archive_mode = off		# enables archiving; off, on, or always
											# (change requires restart)
											
#当启用archive_mode时，可以通过设置 archive_command命令将完成的 WAL 段发送到 归 档存储。
#除用于禁用的off之外，还有两种模式: on和always。
#在普通操作期间，这两 种模式之间 没有区别，但是当设置为always时，WAL 归档器在归档恢复 或者后备模式 下也会被启用。
#在always模式下，所有从归档恢复 的或者用流复制传来的文件将被(再 次)归档。

#archive_mode和archive_command是独立的变量，这样可以在不影响归档模式的前提下修 改archive_command。
#这个参数只能在服务器启动时设置。当wal_level被设置 为minimal时，archive_mode不能被启用。

#archive_command = ''		# command to use to archive a logfile segment
                        # placeholders: %p = path of file to archive
                        #               %f = file name only
                        # e.g. 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
                        
#本地 shell 命令被执行来归档一个完成的 WAL 文件段。字符串中的任何%p被替 换成要被归档的文件的路径名，而%f只被文件名替换(路径名是相
#对于服务器的工作目录，即集簇的数据目录)。如果要在命令里嵌入一个真正的%字符，可以使用%%。
#有一点很重要，该命令只在成功时返回一个零作为退出状态。
#这个参数只能在postgresql.conf文件中或在服务器命令行上设置。除非服务器启动时启 用了archive_mode，否则它会被忽略。
#如果archive_mode被启用时，archive_command是 一个空字符串(默认)，WAL 归档会被临时禁用，但服务器仍会继续累计 WAL 段文件， 
#期待着一个命令被提供。将archive_command设置为一个只返回真但不做任何事的命令 (例如/bin/true或 Windows 上的REM)实际上会禁用归
#档，也会打破归档恢复所需的 WAL 文件链，因此只有在极少数情况下才能用。
                        
#archive_timeout = 0		# force a logfile segment switch after this
												# number of seconds; 0 disables
#archive_command仅在已完成的 WAL 段上调用。因此，如果你的服务器只产生很少的 WAL 流量(或产生流量的周期很长)，那么在事务完成和它被#安全地记录到归档存储之 间将有一个很长的延迟。为了限制未归档数据存在的时间，你可以设 置archive_timeout来强制服务器来周期性地切换到#一个新的 WAL 段文 件。 当这个参数被设置为大于零时，只要从上次段文件切换后过了参数所设置的时间 量，并且已经有过任何数据库活动(包括一#个单一检查点)，服务器将切换到一个新的 段文件(如果没有数据库活动则会跳过检查点)。 注意，由于强制切换而提早关闭的被 归档文件仍然与完整#的归档文件长度相同。因此，使用非常短的archive_timeout是不明 智的 — 它将占用巨大的归档存储。一分钟左右的archive_timeout设置通常#比较合理。 如果你希望数据能被更快地从主服务器上复制下来，你应该考虑使用流复制而不是归 档。如果指定值时没有单位，则以秒为单位。
#这个参数只能在postgresql.conf文件中或 在服务器命令行上设置。												
```

#### 2.4.4 归档恢复

```
# - Archive Recovery -

# These are only used in recovery mode.

#restore_command = ''		# command to use to restore an archived logfile segment
                        # placeholders: %p = path of file to restore
                        #               %f = file name only
                        # e.g. 'cp /mnt/server/archivedir/%f %p'
                        
#用于获取 WAL 文件系列的一个已归档段的本地 shell 命令。这个参数是归档恢复所必需 的，但是对于流复制是可选的。 在该字符串中的任何%f会#被替换为从归档中获得的文件 的名字，并且任何%p会被在服务器上的复制目标路径名替换(该路径名是相对于当前工 作目录的，即集簇的数据目录)。 #任何%r会被包含上一个可用重启点的文件的名字所替 换。 在那些必须被保留用于使得一次恢复变成可重启的文件中，这个文件是其中最早的 一个，
#因此这个信息可以被用来把归档截断为支持从当前恢复重启所需的最小值。 %r通 常只被温备配置(见第 26.2 节)所使用。要嵌入一个真正的%字
#符，需要写成%%。
#很重要的一点是，该命令只有在成功时才返回一个为零的退出状态。 该命令将会被询问 不存在于归档中的文件名，当这样被询问时它必须返回非零。#例子:
#restore_command = 'cp /mnt/server/archivedir/%f "%p"'
#restore_command = 'copy "C:\\server\\archivedir\\%f" "%p"' # Windows
#一个例外是如果该命令被一个信号(不是SIGTERM，它是数据库服务器关闭的一部分)或 者一个 shell 错误(例如命令未找到)终止，则恢复将会中止#并且服务器将不会启动。
#这个参数只能在服务器启动时被设置。

#archive_cleanup_command = ''	# command to execute at every restartpoint

#这个可选参数指定了一个 shell 命令，它将在每一个重启点被执行。 archive_cleanup_command的目的是提供一种清除不再被后备服务器需要的#旧的已归档 WAL 文件的机制。 任何%r会被替换为包含最后一个可用重启点的文件的名 称。 那是使一次恢复变成可重启的所必须被保留的最早的文
#件，并且因此比%r更 早的所有文件可以被安全地移除。 这个信息可以被用来把归档截断为支持从当前 恢复重启所需的最小值。 对于单一后备配置，#pg_archivecleanup模块常常被用 在archive_cleanup_command中，例如:
#archive_cleanup_command = 'pg_archivecleanup /mnt/server/archivedir %r'
#但是注意，如果多个后备服务器正在从同一个归档目录中恢复，你将需要保证只有当任 意服务器都不再需要 WAL 文件时才会删除它们。 #archive_cleanup_command通常被用于 一种温后备配置(见第 26.2 节)中。 要在该命令中嵌入一个真正的%字符，需要写成% %。
#如果该命令返回一个非零退出状态，则将会写出一个警告日志消息。 一个例外是如果该 命令被一个信号或者一个 shell 错误(例如命令未找到)
#终止，则会抛出一个致命错 误。
#这个参数只能在 postgresql.conf 文件中设置或通过服务器命令行的方式。

#recovery_end_command = ''	# command to execute at completion of recovery

#这个参数指定了一个将只在恢复末尾被执行一次的 shell 命令。这个参数 是可选的。 recovery_end_command的目的是为复制或恢复之后的清除#提供一种机 制。 与archive_cleanup_command中相似，任何%r会被替换为包含最后一个可用重启点 的文件的名称。
#如果该命令返回一个非零退出状态，则一个警告日志消息将被写出并且不管怎样该数据 库将继续启动。 一个例外是如果该命令被一个信号或者 #shell 错误(例如命令未找到) 中止，该数据库将不会继续启动。
#这个参数只能在 postgresql.conf 文件中设置或通过服务器命令行的方式。
```

#### 2.4.5 恢复目标

```
# - Recovery Target -

# Set these only when performing a targeted recovery.
#默认情况下，恢复将会一直恢复到 WAL 日志的末尾。下面的参数可以被用来指定一个更早的
#停止点。 在recovery_target、recovery_target_lsn、recovery_target_name、recovery_target_time和recovery_ta 最多只能使#用一个，如果在配置文件中使用了多个，将会产生一个错误。这个参数只能在服 务器启动时设置。

#recovery_target = ''		# 'immediate' to end recovery as soon as a
                        # consistent state is reached
												# (change requires restart)
												
#这个参数指定恢复应该在达到一个一致状态后尽快结束，即尽早结束。在从一个在线备 份中恢复时，这意味着备份结束的那个点。
#在技术上，这是一个字符串参数，但是'immediate'是目前唯一允许的值。

#recovery_target_name = ''	# the named restore point to which recovery will proceed
														# (change requires restart)
#这个参数指定(pg_create_restore_point()所创建)的已命名的恢复点，恢复将进入该 恢复点。

#recovery_target_time = ''	# the time stamp up to which recovery will proceed
														# (change requires restart)
#此参数指定恢复将执行的时间戳。精确的停止点还受到recovery_target_inclusive得影 响。

#recovery_target_xid = ''	# the transaction ID up to which recovery will proceed
													# (change requires restart)
#这个参数指定恢复将进入的事务 ID。记住虽然事务 ID 是在事务开始时顺序分配的，但 是事务可能以不同的数字顺序完成。 
#那些在指定事务之前(也可以包括该事务)提交的 事务将被恢复。精确的停止点也受到recovery_target_inclusive的影响。													
													
#recovery_target_lsn = ''	# the WAL LSN up to which recovery will proceed
													# (change requires restart)
#此参数指定恢复将继续进行的预写日志位置的LSN。精确的停靠点也受 recovery_target_inclusive的影响。 
#使用系统数据类型pg_lsn解析此参数。													
													
#recovery_target_inclusive = on # Specifies whether to stop:
                                # just after the specified recovery target (on)
                                # just before the recovery target (off)
                                # (change requires restart)
#指定我们是否仅在指定的恢复目标之后停止(on)，或者仅在恢复目标之前停止 (off)。 
#适用于recovery_target_lsn、recovery_target_time或 者recovery_target_xid被指定的情况。 
#这个设置分别控制事务是否有准确的目标WAL位 置(LSN)、提交时间或事务ID将被包括在该恢复中。默认值为on。                                
                                
#recovery_target_timeline = 'latest'	# 'current', 'latest', or timeline ID
																			# (change requires restart)
#指定恢复到一个特定的时间线中。该值可以是数字时间线 ID 或特殊值。 值current沿 着与执行基本备份时相同的时间线恢复。
#值latest将恢复到归档中能找到的最新的时间 线，这在一个备用服务器中有用。latest是默认的。
#你通常只需要在复杂的重恢复情况下设置这个参数，在这种情况下你需要返回到一个状 态，该状态本身是在一次时间点恢复之后到达的。
																			
#recovery_target_action = 'pause'	# 'pause', 'promote', 'shutdown'
																	# (change requires restart)
																	
#指定在达到恢复目标时服务器应该立刻采取的动作。默认动作是pause，这表示恢复将会 被暂停。 
#promote表示恢复处理将会结束并且服务器将开始接受连接。 
#最 后，shutdown将在达到恢复目标之后停止服务器。

#使用pause设置的目的是:如果这个恢复目标就是恢复最想要的位置，就允许对数据库执行查询。暂停的状态可以使用pg_wal_replay_resume()继
#续，这会让恢 复终结。 如果这个恢复目标不是想要的停止点，那么关闭服务器，将恢复目标设置改为 一个稍后的目标并且重启以继续恢复。
#要让实例在想要的重放点那里准备好，shutdown设置可以派上用场。 该实例将仍能重放 更多 WAL 记录(并且事实上将不得不重放从下一次它被启动#后最后一个检查点以来的 WAL 记录)。
#注意由于在recovery_target_action被设置为shutdown时，recovery.signal将不会被移 除， 任何后续的启动都将会以立刻关闭为终结，除#非该配置被改变或 者recovery.signal文件被手工移除。
#如果没有设置恢复目标，这个设置没有效果。 如果没有启用hot_standby，pause设置的 动作将和shutdown一样。																	
```

### 2.5 流复制

#### 2.5.1 发送服务器

```
#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

# - Sending Servers -
#这些参数可以在任何发送复制数据给一个或多个后备服务器的服务器上设置。
#主控机总是一 个发送服务器，因此这些参数总是要在主控机上设置。这些参数的角色和含义不会在一个后 备机变成主控机后改变。
# Set these on the primary and on any standby that will send replication data.

#max_wal_senders = 10		# max number of walsender processes
												# (change requires restart)
#指定来自后备服务器或流式基础备份客户端的并发连接的最大数量(即同时运行 WAL 发 送进程的最大数)。 默认值是10。值0意味着禁用复制。
#突然的流客户端断开连接可能 留下一个孤立连接槽(知道达到超时)，因此这个参数应该设置得略高于最大客户端连 接数，这样断开连接的客户端可以#立刻重新连接。 这个参数只能在服务器启动时被设 置。此外，wal_level必须设置为replica或更高级别以允许来自后备服务器的连接。
#当运行备用服务器时，你必须将此参数设置为与主服务器上相同的或更高的值。否则， 备用服务器中将不允许查询。												
												
#max_replication_slots = 10	# max number of replication slots
														# (change requires restart)
														
#指定服务器可以支持的复制槽最大数量。默认值为10。这个参数只 能在服务器启动时设置。
#将它设置为一个比当前已有复制槽要少的值会阻碍服务器启 动。此外，要允许使用复制槽， wal_level必须被设置为replica或 更高。														
#wal_keep_size = 0		# in megabytes; 0 disables

#Specifies the minimum size of past log file segments kept in the pg_wal directory, in case a standby server #needs to fetch them for streaming replication. If a standby server connected to the sending server falls behind #by more than wal_keep_size megabytes, the sending server might remove a WAL segment still needed by the #standby, in which case the replication connection will be terminated. Downstream connections will also #eventually fail as a result. (However, the standby server can recover by fetching the segment from archive, if #WAL archiving is in use.)
#This sets only the minimum size of segments retained in pg_wal; the system might need to retain more segments #for WAL archival or to recover from a checkpoint. Ifwal_keep_size is zero (the default), the system doesn't #keep any extra segments for standby purposes, so the number of old WAL segments available to standby servers is #a function of the location of the previous checkpoint and status of WAL archiving. If this value is specified #without units, it is taken as megabytes. This parameter can only be set in the postgresql.conf file or on the #server command line.

#max_slot_wal_keep_size = -1	# in megabytes; -1 disables

#Specify the max imum size of WAL files that replication slots are allowed to retain in the pg_wal directory at #checkpoint time. #If max_slot_wal_keep_size is -1 (the default), replication slots may retain an unlimited #amount of WAL files. #Otherwise, if restart_lsn of a replication slot falls behind the current LSN by more than #the given size, the #standby using the slot may no longer be able to continue replication due to removal of #required WAL files. You #can see the WAL availability of replication slots in pg_replication_slots. If this #value is specified without #units, it is taken as megabytes. This parameter can only be set in the #postgresql.conf file or on the server #command line.

#wal_sender_timeout = 60s	# in milliseconds; 0 disables

#中断那些停止活动超过这个时间量的复制连接。这对发送服务器检测一个费用服务器崩 溃或网络中断有用。 
#如果指定值时没有单位，则以毫秒为单位。默认值是 60 秒。值0 将禁用超时机制。
#对于跨越多个地理位置分布的集群，每个位置使用不同的值可以为集群管理带来更大的 灵活性。 对于具有低延迟网络连接的备用服务器，较小的值有#助于更快的故障检 测; 对于位于远端位置的具有高延迟网络连接的备用服务器，较大的值有助于更好地判 断备用服务器的健康状况。

#track_commit_timestamp = off	# collect timestamp of transaction commit
															# (change requires restart)
#记录事务的提交时间。这个参数只能在postgresql.conf 文件中或在服务器命令行上设 置。默认值是off。														
```

#### 2.5.2 主服务器

```
# - Primary Server -

# These settings are ignored on a standby server.

#synchronous_standby_names = ''	# standby servers that provide sync rep
                                # method to choose sync standbys, number of sync standbys,
                                # and comma-separated list of application_name
                                # from standby(s); '*' = all
				
#这个参数指定一个支持同步复制的后备服务器的列表。				


#vacuum_defer_cleanup_age = 0	# number of xacts by which cleanup is delayed

#指定VACUUM和HOT更新在清除死亡行版本之前，应该推迟多久(以事务数量计)。默认值 是零个事务，表示死亡行版本将被尽可能快地清除，即当它们#不再对任何打开的事务可 见时尽快清除。在一个支持热后备服务器的主服务器上，你可能希望把这个参数设置为 一个非零值，。这允许后备机上的查#询有更多时间来完成而不会由 于先前的行清除产生冲突。但是，由于该值是用在主服务器上发生的写事务的数目衡量 的，很难预测对后备机查询可用#的附加时间到底是多少。这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。
#你也可以考虑设置后备服务器上的hot_standby_feedback作为使用这个参数的一种替代 方案。
```

#### 2.5.3 备服务器

```
# - Standby Servers -

# These settings are ignored on a primary server.

#primary_conninfo = ''			# connection string to sending server

#指定备用服务器用来连接发送服务器的连接字符串。

#primary_slot_name = ''			# replication slot on sending server

#有选择地指定通过流复制连接到发送服务器时使用一个现有的复制槽来控制上游节点上 的资源移除。 
#这个参数只能在服务器启动时被设置。如果没有设 置primary_conninfo则这个设置无效。


#promote_trigger_file = ''		# file name whose presence ends recovery

#指定一个触发器文件，该文件的存在会结束后备机中的恢复。即使这个值没有被设置， 你也能够使用pg_ctl promote或调用pg_promote来提升后
#备机。 这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。

#hot_standby = on			# "off" disallows queries during recovery
											# (change requires restart)
#指定在恢复期间，你是否能够连接并运行查询，默认值是on。这 个参数只能在服务器启动时设置。它只在归档恢复期间或后备机模式下才有效。											
											
#max_standby_archive_delay = 30s	# max delay before canceling queries
                                  # when reading WAL from archive;
                                  # -1 allows indefinite delay
                                  
#当热后备机处于活动状态时，这个参数决定取消那些与即将应用的 WAL 项冲突的后备机 查询之前，后备服务器应该等待多久。 
#当 WAL 数据被从 WAL 归档(并且因此不是当前的 WAL)时，max_standby_archive_delay可以应用。 
#如果指 定值时没有单位，则以毫秒为单位。默认值是 30 秒。值 -1 允许后备机一直等到冲突 查询结束。
#这个参数只能在postgresql.conf文件中或在服务器命令行上设置。
#注意，max_standby_archive_delay与取消之前一个查询能够运行的最长时间不同;
#它表 示应用任何一个 WAL 段数据能够被允许的最长总时间。因此，如果一个查询早于 WAL 段导致了显著的延迟，后续冲突查询将只有更少的时间。

#max_standby_streaming_delay = 30s	# max delay before canceling queries
                                    # when reading streaming WAL;
                                    # -1 allows indefinite delay
                                    
#当热后备机处于活动状态时，这个参数决定取消那些与即将应用的 WAL 项冲突的后备机 查询之前，后备服务器应该等待多久。 
#当 WAL 数据正在通过 流复制被接收时，max_standby_streaming_delay可以应用。 如果指定值时没有单位， 则以毫秒为单位。默认值是 #30 秒。值 -1 允许后备机一直等到冲突查询结束。 这个参 数只能在postgresql.conf文件中或在服务器命令行上设置。
#注意，max_standby_streaming_delay与取消之前一个查询能够运行的最长时间不同;它 表示在从主服务器接收到 WAL 数据并立刻应用它能够被#允许的最长总时间。因此，如果一个查询导致了显著的延迟，后续冲突查询将只有更少的时间，直到后备服务器再次赶 上进度。

#wal_receiver_create_temp_slot = off	# create temp slot if primary_slot_name
																			# is not set
#wal_receiver_status_interval = 10s	# send replies at least this often
																		# 0 disables
																		
#指定在后备机上的 WAL 接收者进程向主服务器或上游后备机发送有关复制进度的信息的 最小频度，它可以使用pg_stat_replication视图看到。 #后备机将报告它已经写入的上 一个预写式日志位置、它已经刷到磁盘的上一个位置以及它已经应用的最后一个位置。 这个参数的值是报告之间的最大#时间量。每次写入或刷出位置改变时会发送状态更新， 或者至少按这个参数的指定的频度发送。因此，应用位置可能比真实位置略微滞后。 如 果指
#定值时没有单位，则以秒为单位。默认值是 10 秒。将这个参数设置为零将完全禁 用状态更新。这个参数只能在postgresql.conf文件中或在服务
#器命令行上设置。							

#hot_standby_feedback = off		# send info from standby to prevent
															# query conflicts
															
#指定一个热后备机是否将会向主服务器或上游后备机发送有关于后备机上当前正被执行 的查询的反馈。这个参数可以被用来排除由于记录清除导致的查#询取消，但是可能导致 在主服务器上用于某些负载的数据库膨胀。反馈消息的发送频度不会高于每 个wal_receiver_status_interval周期发送
#一次。默认值是off。这个参数只能 在postgresql.conf文件中或在服务器命令行上设置。
#如果使用级联复制，反馈将被向上游传递直到它最后到达主服务器。后备机在接收到反 馈之后除了传递给上游不会做任何其他操作。
#这个设置不会覆盖主服务器上的old_snapshot_threshold的行为，后备服务器上一个超 过了主服务器年龄阈值的快照可能会变得不可用，导致后
#备服务器上事务的取消。这是 因为old_snapshot_threshold是为了对死亡行能够存在的时间给出一个绝对限制，不然 就会因为一个后备服务器的#配置而被违背。															
															
#wal_receiver_timeout = 60s		# time that receiver waits for
                              # communication from primary
                              # in milliseconds; 0 disables
#中止处于非活动状态超过这个时间量的复制链接。这对于正在接收的后备服务器检测主 服务器崩溃或网络断开有用。 如果指定值时没有单位，则以毫#秒为单位。默认值是 60 秒。 这个参数只能在postgresql.conf文件中或者服务器命令行中设置。                              
                              
#wal_retrieve_retry_interval = 5s	# time to wait before retrying to
																	# retrieve WAL after a failed attempt
																	
#指定当从任何来源(流复制、本地pg_wal或者 WAL 归档)都得不到 WAL 数据时，后备服 务器应该等待多久才去重新尝试获取 WAL 数据。 如果指
#定值时没有单位，则以毫秒为单 位。默认值是 5 秒。 这个参数只能在postgresql.conf文件或者服务器命令行中设置。
#这个参数对恢复中的节点需要为新 WAL 数据可用等待多少时间的配置有用。 例如，在 归档恢复中，通过减小这个参数的值可以让恢复更积极地检测#新的 WAL 日志文件。在一 个 WAL 活动较低的系统上，增加这个参数的值可以减 少访问 WAL 归档所必需的请求数 量，这对于例如云环境是有用
#的，在其中 对于基础设施的访问时间也是被考虑的。																	
																	
#recovery_min_apply_delay = 0		# minimum delay for applying changes during recovery


```

#### 2.5.4 订阅者

```
# - Subscribers -

# These settings are ignored on a publisher.

#max_logical_replication_workers = 4	# taken from max_worker_processes
					# (change requires restart)
					
#指定逻辑复制工作者的最大数目。这同时包括应用工作者和表同步工作者。逻辑复制工作者是从max_worker_processes定义的池中取出的。默认4。					
#max_sync_workers_per_subscription = 2	# taken from max_logical_replication_workers

#每个订阅的同步工作者的最大数目。这个参数控制订阅初始化期间或者新表增加时的初 始数据拷贝的并行度。
#当前，每个表只能有一个同步工作者。 同步工作者是从max_logical_replication_workers定义的池中取出的。 默认值为2。
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
#允许或禁止查询规划器使用位图扫描计划类型。
#enable_gathermerge = on
#启用或者禁用查询规划器对收集归并计划类型的使用。
#enable_hashagg = on
#允许或禁用查询规划器使用哈希聚集计划类型。
#enable_hashjoin = on
#允许或禁止查询规划器使用哈希连接计划类型。
#enable_incremental_sort = on
#enable_indexscan = on
#允许或禁止查询规划器使用索引扫描计划类型
#enable_indexonlyscan = on
#允许或禁止查询规划器使用只用索引扫描计划类型
#enable_material = on
#允许或者禁止查询规划器使用物化。它不可能完全禁用物化，但是关闭这个变量将阻止 规划器插入物化节点，除非为了保证正确性。默认值是on。
#enable_memoize = on
#enable_mergejoin = on
#允许或禁止查询规划器使用归并连接计划类型。
#enable_nestloop = on
#允许或禁止查询规划器使用嵌套循环连接计划。它不可能完全禁止嵌套循环连接，但是 关闭这个变量将使得规划器尽可能优先使用其他方法。
#enable_parallel_append = on
#允许或禁止查询规划器使用并行追加计划类型。
#enable_parallel_hash = on
#允许或禁止查询规划器对并行哈希使用哈希连接计划类型。如果哈希连接计划也没有启 用，这个参数没有效果。
#enable_partition_pruning = on
#允许或者禁止查询规划器从查询计划中消除一个分区表的分区。这也控制着规划器产生允许执行器在查询执行期间移除(忽略)分区的查询计划的能力。
#enable_partitionwise_join = off
#允许或者禁止查询规划器使用面向分区的连接，这使得分区表之间的连接以连接匹配的 分区的方式来执行。面向分区的连接当前只适用于连接条件包括#所有分区键的情况，连接条件必须是相同的数据类型并且子分区集合要完全匹配。由于面向分区的连接规划在 规划期间会使用可观的CPU时间和内存，#所以默认值为off。
#enable_partitionwise_aggregate = off
#允许或者禁止查询规划器使用面向分区的分组或聚集，这使得在分区表上的分组或聚集 可以在每个分区上分别执行。如果GROUP BY子句不包括分区
#键，只有部分聚集能够以基 于每个分区的方式执行，并且finalization必须最后执行。由于面向分区的分组或聚集 在规划期间会使用可观的CPU时#间和内存，所以默认值为off。
#enable_seqscan = on
#允许或禁止查询规划器使用顺序扫描计划类型。它不可能完全禁止顺序扫描，但是关闭 这个变量将使得规划器尽可能优先使用其他方法。
#enable_sort = on
#允许或禁止查询规划器使用显式排序步骤。它不可能完全禁止显式排序，但是关闭这个 变量将使得规划器尽可能优先使用其他方法。
#enable_tidscan = on
#允许或禁止查询规划器使用TID扫描计划类型。
```

#### 2.6.2 规划器代价常量

```
# - Planner Cost Constants -

#seq_page_cost = 1.0			# measured on an arbitrary scale

#设置规划器计算一次顺序磁盘页面抓取的开销。默认值是1.0。 
#通过设置同名的表空间参数，这个值可以重写为一个特定的表空间。参阅ALTER TABLESPACE。  
 

#random_page_cost = 4.0			# same scale as above

#设置规划器对一次非顺序获取磁盘页面的代价估计。默认值是 4.0。
#通过把表和索引放 在一个特殊的表空间(要设置该表空间的同名参数)中可以覆盖这个值(见ALTER TABLESPACE)。
#减少这个值(相对于seq_page_cost)将导致系统更倾向于索引扫描;提高它将让索引扫 描看起来相对更昂贵。
#你可以一起提高或降低两个值来改变磁盘 I/O 代价相对于 CPU 代 价的重要性，后者由下列参数描述。

#对磁盘存储的随机访问通常比顺序访问要贵不止四倍。但是，由于对磁盘的大部分随机 访问(例如被索引的读取)都被假定在高速缓冲中进行，所以使
#用了一个较低的默认值(4.0)。默认值可以被想成把随机访问建模为比顺序访问慢 40 倍，而期望 90% 的随 机读取会被缓存。
#如果你相信 90% 的缓冲率对你的负载是一个不正确的假设，你可以增加 random_page_cost 来更好的反映随机存储读取的真正代价。
#相应地，如果你的数据可以 完全放在高速缓存中(例如当数据库小于服务器总内存时)，
#降低 random_page_cost 可能是合适的。为具有很低的随机读取代价的存储(例如固态驱动器)采用较低的 random_page_cost 值可能更好。

#cpu_tuple_cost = 0.01			# same scale as above

#设置规划器对一次查询中处理每一行的代价估计。

#cpu_index_tuple_cost = 0.005		# same scale as above

#设置规划器对一次索引扫描中处理每一个索引项的代价估计。

#cpu_operator_cost = 0.0025		# same scale as above

#设置规划器对于一次查询中处理每个操作符或函数的代价估计。

#parallel_setup_cost = 1000.0	# same scale as above

#设置规划器对启动并行工作者进程的代价估计。默认是 1000。

#parallel_tuple_cost = 0.1		# same scale as above

#设置规划器对于从一个并行工作者进程传递一个元组给另一个进程的代价估计。默认是 0.1。

#min_parallel_table_scan_size = 8MB

#为必须扫描的表数据量设置一个最小值，扫描的表数据量超过这一个值才会考虑使用并行扫描。
#对于并行顺序扫描，被扫描的表数据量总是等于表的尺寸，但是在使用 索引时，被扫描的表数据量通常会更小。 
#如果指定值时没有单位，则以块为单位， 即BLCKSZ字节，通常为8kB。默认值是(8MB)。


#min_parallel_index_scan_size = 512kB

#为必须扫描的索引数据量设置一个最小值，扫描的索引数据量超过这一个值时才会考虑 使用并行扫描。
#注意并行索引扫描通常并不会触及整个索引，它是规划器认为该扫描会 实际用到的相关页面的数量。 
#如果指定值时没有单位，则以块为单位，即BLCKSZ字节， 通常为8kB。默认值是(512kB)。

#effective_cache_size = 4GB

#设置规划器对一个单一查询可用的有效磁盘缓冲区尺寸的假设。 
#这个参数会被考虑在使 用一个索引的代价估计中，更高的数值会使得索引扫描更可能被使用，更低的数值会使 得顺序扫描更可能被使用。 
#在设置这个参数时，你还应该考虑PostgreSQL的共享缓冲区以及将被用于数据文件的内核磁盘缓冲区，尽管有些数据可能在两个地方都存在。 
#另外，还要考虑预计在不同表上的并发查询数目，因为它们必须共享可用的空 间。 
#这个参数对PG分配的共享内存尺寸没有影响，它也不会保留内核磁盘缓冲，只用于估计的目的。系统也不会假设在查询之间数据会保留在磁盘缓冲中。 
#如果指定值时没有单位，则以块为单位，即BLCKSZ字节，通常为8kB。 默认值是(4GB)。(如果BLCKSZ不是8kB，默认值会按比例缩放它。)

#jit_above_cost = 100000		# perform JIT compilation if available
                            # and query more expensive than this;
                            # -1 disables
#设置激活JIT编译的查询代价，如果查询代价超过这个值就会激活JIT编译。
#执行JIT会消耗一些规划时间，但是能够加速查询执行。将这个值 设置为-1会禁用JIT编译。默认值是100000。					
					
#jit_inline_above_cost = 500000		# inline small functions if query is
																	# more expensive than this; -1 disables
#设置JIT编译尝试内联函数和操作符的查询代价阈值，如果查询代价超过这个值，JIT编 译就会尝试内联。
#内联会增加规划时间，但是可以提高执行速度。将这个参数设置成小 于jit_above_cost是没有意义的。将这个参数设置为-1会禁用内联。
																	
#jit_optimize_above_cost = 500000	# use expensive JIT optimizations if
                                  # query is more expensive than this;
                                  # -1 disables
                                  
#设置JIT编译应用优化的查询代价阈值，如果查询代价超过这个值，JIT编译就会应用开 销较大的优化。
#这类优化会增加规划时间，但是更能够改进执行速度。
#将这个参数设置 成小于jit_above_cost是没有意义的，并且将它设置成大于jit_inline_above_cost也未 必有益。
#将这个参数设置为-1会禁用开销较大的优化。默认值是500000。                                  
```

#### 2.6.3 遗传查询优化

```
# - Genetic Query Optimizer -

#遗传查询规划器(GEQO)是一种使用启发式搜索来进行查询规划的算法。
#它可以降低 对于复杂查询(连接很多表的查询)的规划时间，但是代价是它产生的计划有时候要差于使 用穷举搜索算法找到的计划。

#geqo = on
#允许或禁止遗传查询优化。默认是启用。在生产环境中通常最好不要关闭 它。

#geqo_threshold = 12

#只有当涉及的FROM项数量至少有这么多个的时候，才使用遗传查询优化(注意一个FULL OUTER JOIN只被计为一个FROM项)。默认值是 12。
#对于更简单的查询，通常会使用普通 的穷举搜索规划器，
#但是对于有很多表的查询穷举搜索会花很长时间，通常比执行一个次优的计划带来的惩罚值还要长。
#因此，在查询尺寸上的一个阈值是管理 GEQO 使用的一种方便的方法。

#geqo_effort = 5			# range 1-10

#控制 GEQO 里规划时间和查询规划的有效性之间的平衡。
#大的数值增加花在进行查询规划上面的时间，但是也很可能会提高选中更有效的查询规划的几率。 
#控制 GEQO 中规划时间和查询计划质量之间 的折中。这个变量必须是位于 1 到 10 之间的一个整数。默认值是 5。
#更大的值会增加花在查询规划上的时间，但是同时也增加了选择一个高效查询计划的可能性。
#geqo_effort实际并不直接做任何事情;它只是被用来计算其他影响 GEQO 行为的变量的默认值（geqo_pool_size，geqo_generations）。

#geqo_pool_size = 0			# selects default based on effort

#控制 GEQO 使用的池尺寸，它就是遗传种群中的个体数目。它必须至少为 2，且有用的 值通常在 100 到 1000 之间。
#如果它被设置为零(默认设置)则会基于geqo_effort和 查询中表的数量选择一个合适的值。

#geqo_generations = 0			# selects default based on effort

#控制 GEQO 使用的池尺寸，它就是遗传种群中的个体数目。它必须至少为 2，且有用的 值通常在 100 到 1000 之间。
#如果它被设置为零(默认设置)则会基于geqo_effort和 查询中表的数量选择一个合适的值。


#geqo_selection_bias = 2.0		# range 1.5-2.0

#控制 GEQO 使用的选择偏好。选择偏好是种群中的选择压力。

#geqo_seed = 0.0			# range 0.0-1.0

#控制 GEQO 使用的随机数生成器的初始值，随机数生成器用于在连接顺序搜索空间中选择随机路径。
#变化该值会改变被探索的连接路径集合， 并且可能导致找到一个更好或更差的路径。
```

#### 2.6.4 其他规划器选项

```
# - Other Planner Options -

#default_statistics_target = 100	# range 1-10000

#为没有通过ALTER TABLE SET STATISTICS设置列相关目标的表列设置默认统计目标。
#更大 的值增加了需要做ANALYZE的时间，但是可能会改善规划器的估计质量。默认值是 100。

#constraint_exclusion = partition	# on, off, or partition

#控制查询规划器对表约束的使用，以优化查询。 constraint_exclusion的允许值 是on(对所有表检查约束)、off(从不检查约束)
#和partition(只对继承的子表 和UNION ALL子查询检查约束)。 partition是默认设置。它通常与传统的继承树一起使 用来提高性能。

#cursor_tuple_fraction = 0.1		# range 0.0-1.0

#设置规划器对将被检索的一个游标的行的比例的估计。默认值是 0.1。更小的值使得规 划器偏向为游标使用“快速开始”计划，它将很快地检索前几行#但是可能需要很长时间 来获取所有行。更大的值强调总的估计时间。最大设置为 1.0，游标将和普通查询完全 一样地被规划，只考虑总估计时间并且#不考虑前几行会被多快地返回。

#from_collapse_limit = 8

#如果生成的FROM列表不超过这么多项，规划器将把子查询融合到上层查询。较小的值可 以减少规划时间，但是可能会生成较差的查询计划。
#将这个值设置为geqo_threshold或更大，可能触发使用 GEQO 规划器，从而产生非最优 计划。

#jit = on				# allow JIT compilation

#PostgreSQL是否可以使用JIT编译。

#join_collapse_limit = 8		# 1 disables collapsing of explicit
														# JOIN clauses
					
#如果得出的列表中不超过这么多项，那么规划器将把显式JOIN(除了FULL JOIN)结构重 写到 FROM项列表中。
#较小的值可减少规划时间，但是可能会生成差些的查询计划。
#默认情况下，这个变量被设置成和from_collapse_limit相同， 这样适合大多数使用。
#把它设置为 1 可避免任何显式JOIN的重排序。因此查询中指定的显式连接顺序就是关系被连接的实际顺序。
#因为查询规划器并不是总能 选取最优的连接顺序，高级用户 可以选择暂时把这个变量设置为 1，然后显式地指定他们想要的连接顺序。

#将这个值设置为geqo_threshold或更大，可能触发使用 GEQO 规划器，从而产生非最优 计划。

#plan_cache_mode = auto			# auto, force_generic_plan or
														# force_custom_plan
														
#准备语句(显式准备或隐式生成的，例如 PL/pgSQL)可以使用自定义或通用计划执行。
#使用其特定的参数值集为每个执行重新生成自定义计划，而通用计划不依赖于参数值，并且可以在执行中重复使用。
#因此，使用通用计划可以节省计划时间，但如果理想计划严重依赖参数值，则通用计划可能效率低下。 
#这些选项之间的选择通常是自 动进行的，但可以通过plan_cache_mode覆盖它。 
#这个设置是在执行缓存计划时考虑， 而不是在准备计划时考虑。 更多信息请参阅 PREPARE.														
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

#PostgreSQL支持多种方法来记录服务器消息，包括stderr、csvlog和syslog。
#在 Windows 上还支持eventlog。设置这个参数为一个由想要的日志目的地的列表，之间用 逗号分隔。

# This is used when logging to stderr:
#logging_collector = off		# Enable capturing of stderr and csvlog
                            # into log files. Required to be on for
                            # csvlogs.
                            # (change requires restart)
#这个参数启用日志收集器，它是一个捕捉被发送到stderr的日志消息的后台进程，并且 它会将这些消息重定向到日志文件中。

# These are only used if logging_collector is on:
#log_directory = 'log'			# directory where log files are written,
					# can be absolute or relative to PGDATA
					
#当logging_collector被启用时，这个参数决定日志文件将被在哪个目录下创建。
#它可以 被指定为一个绝对路径，也可以被指定为一个相对于集簇数据目录的相对路径。

#log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'	# log file name pattern,
					# can include strftime() escapes

#设置被创建的日志文件的文件名。
					
#log_file_mode = 0600			# creation mode for log files,
													# begin with 0 to use octal notation
#设置被创建的日志文件的文件权限。													
													
#log_rotation_age = 1d			# Automatic rotation of logfiles will
														# happen after that time.  0 disables.

#决定使用一个单个日志文件的最大时间量，之后将创立一个新的日志文件。	
#将这个参数设置为零将禁用基于时间的新日志文件创建。
														
#log_rotation_size = 10MB		# Automatic rotation of logfiles will
                            # happen after that much log output.
                            # 0 disables.
#这个参数决定一个个体日志文件的最大尺寸。 当这些数据量被发送到一个日志文件后，将创建一个新的日志文件。                            
                            
#log_truncate_on_rotation = off		# If on, an existing log file with the
                                  # same name as the new log file will be
                                  # truncated rather than appended to.
                                  # But such truncation only occurs on
                                  # time-driven rotation, not on restarts
                                  # or size-driven rotation.  Default is
                                  # off, meaning append to existing files
                                  # in all cases.
#当出现存在文件时，是否截断（覆盖而不是追加）

# These are relevant when logging to syslog:
#syslog_facility = 'LOCAL0'

#当启用了向syslog记录时，这个参数决定要使用的syslog“设备”。你可以 在LOCAL0、LOCAL1、LOCAL2、LOCAL3、LOCAL4、 LOCAL5、#LOCAL6、LOCAL7中选择，默认 值是LOCAL0。还请参阅系统的syslog守护进程的文档。

#syslog_ident = 'postgres'

#当启用了向syslog记录时，这个参数决定用来标识syslog中的PostgreSQL消息的程序 名。

#syslog_sequence_numbers = on

#记录到syslog是否加序列号

#syslog_split_messages = on

#当启用把日志记录到syslog时，这个参数决定消息如何送达 syslog。当设置为 on(默 认)时，消息会被分成行，并且长的行也会被划分以便能够放#到 1024 字节中，这是传统 syslog 实现一种典型的尺寸限制。当设置为 off 时，PostgreSQL 服务器日志消息 会被原样送达 syslog 服务，#而处理可能的大体量消息的任务由 syslog 服务负责。

# This is only relevant when logging to eventlog (Windows):
# (change requires restart)
#event_source = 'PostgreSQL'

#当启用了向事件日志记录时，这个参数决定用来标识日志中PostgreSQL消息的程序名。 默认值是PostgreSQL。
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
#控制哪些消息级别

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

#控制哪些导致一个错误情况的 SQL 语句被记录在服务器日志中。任何指定 严重级 别 或更高级别的消息的当前 SQL 语句将被包括在日志项中。

#log_min_duration_statement = -1	# -1 is disabled, 0 logs all statements
					# and their durations, > 0 logs only
					# statements running at least this number
					# of milliseconds

#如果语句运行至少指定的时间量，将导致记录每一个这种完成的语句的持续时间。如果 指定值时没有单位，则以毫秒为单位。
#将这个参数设置为零将打印所有语句的执行时 间。 设置为 -1 (默认值)将停止记录语句持续时间。
#例如，如果你设置它为250ms， 那么所有运行 250ms 或更久的 SQL 语句将被记录。启用这个参数可以有助于追踪应用 中未优化的查询

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
#设置所有语句都被记录的交易的部分，以及由于其他原因而记录的语句。它适用于每个新事务，而不考虑其语句的持续时间。 
#默认值是 0，表示不记录任何额外事务的语句。设置这个为1记录所有交易的所有语句。 
#log_transaction_sample_rate 有助于跟踪交易样本。
					
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

#导致检查点和重启点被记录在服务器日志中。一些统计信息也被包括在日志消息中，包 括写入缓冲区的数据和写它们所花的时间。

#log_connections = off

#导致每一次尝试对服务器的连接被记录，客户端认证的成功完成也会被记录。 

#log_disconnections = off

#导致会话终止被记录。日志输出提供的信息类似于 log_connections，不过还外加会话 的持续时间。 

#log_duration = off

#导致每一个完成的语句的持续时间被记录。默认值是off。

#log_error_verbosity = default		# terse, default, or verbose messages

#控制为每一个被记录的消息要写入到服务器日志的细节量。

#log_hostname = off

#控制是否现实hostname

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
					
#日志记录前缀

#log_lock_waits = off			# log lock waits >= deadlock_timeout

#控制当一个会话为获得一个锁等到超过deadlock_timeout时，是否要产生一个日志消息。这有助于决定是否所等待造成了性能低下。默认值是off。

#log_recovery_conflict_waits = off	# log standby recovery conflict waits
																		# >= deadlock_timeout
#log_parameter_max_length = -1		# when logging statements, limit logged
                                  # bind-parameter values to N bytes;
                                  # -1 means print in full, 0 disables
#log_parameter_max_length_on_error = 0	# when logging an error, limit logged
                                        # bind-parameter values to N bytes;
                                        # -1 means print in full, 0 disables
#log_statement = 'none'			# none, ddl, mod, all

#控制哪些 SQL 语句被记录。
#ddl记录所有数据定义语句，例如CREATE、ALTER和 DROP语句。
#mod记录所有ddl语 句，外加数据修改语句例如INSERT, UPDATE、DELETE、TRUNCATE, 和COPY FROM。 
#如果PREPARE、EXECUTE和 EXPLAIN ANALYZE包含合适类型的命令，它们也会被记录。

#log_replication_commands = off

#导致每一个复制命令都被记录在服务器日志中,默认值是off

#log_temp_files = -1			# log temporary files equal or larger
                          # than the specified size in kilobytes;
                          # -1 disables, 0 logs all temp files
#控制记录临时文件名和尺寸。临时文件可以被创建用来排序、哈希和存储临时查询结果。
#如果启用这个设置，当每一个临时文件被删除时都会产生一个日志项。 
#一个零值 记录所有临时文件信息，而正值只记录尺寸大于或等于指定数据量的文件。如果指定值时没有单位，则以KB为单位。
#默认设置为 -1，它禁用这种记录。                         
                          
log_timezone = 'PRC'

#设置在服务器日志中写入的时间戳的时区。和TimeZone不同，这个值是集簇范围的，因 此所有会话将报告一致的时间戳。
```

#### 2.7.4 进程标题

```
#------------------------------------------------------------------------------
# PROCESS TITLE
#------------------------------------------------------------------------------

#cluster_name = ''			# added to process titles if nonempty
												# (change requires restart)
												
#为不同目的设置标识这个数据库集群(实例)的名称。此集群名称出现在该集群中所有服务器进程的进程标题中。 
#此外，它还是备用连接的默认应用名称。(参见 synchronous_standby_names.)
#这个名称可以是任何长度不超过NAMEDATALEN个字符(在标准编译中是 64字符)的任何 字符串。
#只有可打印的 ASCII 字符能被用在cluster_name值中。其他字符将被替换为问 号(?)。
#如果这个参数被设置为空字符串''(也是默认值)，将不会显示名称。		


#update_process_title = on

#启用进程标题更新，每次服务器接收到一个新的 SQL 命令时都更新进程标题。
#在大部 分平台上这个设置默认为on，但是由于 Windows 上更新进程标题的开销更大，所以在 Windows 这个设置默认为off。

```

### 2.8 统计信息

#### 2.8.1 查询索引统计收集器

```
#------------------------------------------------------------------------------
# STATISTICS
#------------------------------------------------------------------------------

#这些参数控制服务器范围的统计数据收集特性。当统计收集被启用时，被产生的数据可以通 过pg_stat和pg_statio系统视图族访问

# - Query and Index Statistics Collector -

#track_activities = on

#启用对每个会话的当前执行命令的信息收集，还有命令开始执行的时间。这个参数默认 为打开。
#注意即使被启用，这些信息也不是对所有用户可见，只有超级用户和拥有报告信息的会话的用户可见，因此它不会表现为一个安全风险。

#track_activity_query_size = 1024	# (change requires restart)

#为每个活动会话指定存储当前执行命令的文本所保留的内存量，它们被用于pg_stat_activity.query域。 
#如果指定值时没有单位，则以字节为单位。默认值是 1024字节。

#track_counts = on

#启用在数据库活动上的统计收集。这个参数默认为打开，因为自动清理守护进程需要被 收集的信息。

#track_io_timing = off

#启用对系统 I/O 调用的计时。这个参数默认为关闭，因为它将重复地向操作系统查询当前时间，这会在某些平台上导致显著的负荷。
#你可以使用pg_test_timing工具来度量你的系统中计时的开销。
#I/O 计时信息被显示在pg_stat_database中、当BUFFERS选项被使 用时的EXPLAIN输出中以及pg_stat_statements中。

#track_wal_io_timing = off

#作用同track_io_timing，启用WAL调用计时。I/O 计时信息被显示在pg_stat_wal

#track_functions = none			# none, pl, all

#启用跟踪函数调用计数和用时。指定pl只跟踪过程语言函数，指定all还会跟踪 SQL 和 C 语言函数。默认值是none，它禁用函数统计跟踪。

#stats_temp_directory = 'pg_stat_tmp'

#设置存储临时统计数据的目录。这可以是一个相对于数据目录的路径或一个绝对路径。
#默认值是pg_stat_tmp。在一个基于 RAM 的文件系统上指明这个参数将降低物理 I/O 需求，并且提高性能。
```

#### 2.8.2 统计信息监控

```
# - Monitoring -
#对每个查询，向服务器日志里输出相应模块的性能统计。这是一种粗糙的分析工具。类 似于 Unix 的getrusage()系统功能。#log_statement_stats报告总的语句统计，而其它 的报告针每个模块的统计。
#log_statement_stats不能和 其它任何针对每个模块统计的 选项一起启用。
#所有这些选项都是默认禁用的。只有超级用户可以更改这个设置。

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
					
#控制服务器是否运行自动清理启动器后台进程。默认为开启， 不过要自动清理正常工作还需要启用track_counts。 
#该参数只能在postgresql.conf文件或服务器命令行中设 置， 不过，通过更改表存储参数可以为表禁用自动清理。
#注意即使该参数被禁用，系统也会在需要防止事务ID回卷时发起清理进程。

#autovacuum_max_workers = 3		# max number of autovacuum subprocesses
															# (change requires restart)
															
#指定能同时运行的自动清理进程(除了自动清理启动器之外)的最大数量

#autovacuum_naptime = 1min		# time between autovacuum runs

#指定自动清理在任意给定数据库上运行的最小延迟。在每一轮中后台进程检查数据库并 根据需要为数据库中的表发出VACUUM和ANALYZE命令。
#如果指定值时没有单位，则以秒 为单位。默认值为1分钟(1min)。

#autovacuum_vacuum_threshold = 50	# min number of row updates before
																	# vacuum
																	
#指定能在一个表上触发VACUUM的被更新或被删除元组的最小数量。默认值为50 个元组。

#autovacuum_vacuum_insert_threshold = 1000	# min number of row inserts
                                            # before vacuum; -1 disables insert
                                            # vacuums
#指定能在一个表上触发VACUUM的被插入元组的最小数量。默认值为1000 个元组。					
					
#autovacuum_analyze_threshold = 50	# min number of row updates before
																		# analyze
																		
#指定能在一个表上触发ANALYZE的被插入、被更新或被删除元组的最小数量。默认值为50 个元组。

#autovacuum_vacuum_scale_factor = 0.2	# fraction of table size before vacuum

#指定一个表尺寸的分数，在决定是否触发VACUUM时将它加到autovacuum_vacuum_threshold上。默认值为0.2(表尺寸的20%)

#autovacuum_vacuum_insert_scale_factor = 0.2	# fraction of inserts over table
																							# size before insert vacuum
																							
#指定一个表上插入的分数，在决定是否触发VACUUM时将它加到autovacuum_vacuum_threshold上。默认值为0.2(表尺寸的20%)

#autovacuum_analyze_scale_factor = 0.1	# fraction of table size before analyze

#指定一个表尺寸的分数，在决定是否触发ANALYZE时将它加 到autovacuum_analyze_threshold上。默认值为0.1(表尺寸的10%)

#autovacuum_freeze_max_age = 200000000	# maximum XID age before forced vacuum
																				# (change requires restart)
#指定在一个VACUUM操作被强制执行来防止表中事务ID回卷之前，一个表 的pg_class.relfrozenxid域能保持的最大年龄(事务的)。
#注意即便自动清理被禁用， 系统也将发起自动清理进程来阻止回卷。
#清理也允许从pg_xact子目录中移除旧文件，这也是为什么默认值被设置为较低的2亿事 务。 

#autovacuum_multixact_freeze_max_age = 400000000	# maximum multixact age
                                                  # before forced vacuum
                                                  # (change requires restart)
#指定在一个VACUUM操作被强制执行来防止表中多事务ID回卷之前，一个表的pg_class.relminmxid域能保持的最大年龄(多事务的)。
#注意即便自动清理被禁用， 系统也将发起自动清理进程来阻止回卷。
#清理多事务也允许从pg_multixact/members和pg_multixact/offsets子目录中移除旧文件，这也是为什么默认值设置为较低的400万多事务。                                                  

#autovacuum_vacuum_cost_delay = 2ms	# default vacuum cost delay for
                                    # autovacuum, in milliseconds;
                                    # -1 means use vacuum_cost_delay
                                    
#指定用于自动VACUUM操作中的代价延迟值。如果指定-1(默认值)，则使 用vacuum_cost_delay值。 
#如果指定值时没有单位，则以毫秒为单位。默认值为2毫秒。               

#autovacuum_vacuum_cost_limit = -1	# default vacuum cost limit for
                                    # autovacuum, -1 means use
                                    # vacuum_cost_limit
#指定用于自动VACUUM操作中的代价限制值。如果指定-1(默认值)，则使用vacuum_cost_limit值。
#注意该值被按比例地分配到运行中的自动清理工作者上(如果 有多个)，因此每一个工作者的限制值之和不会超过这个变量中的值。                                    
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
#客户端可以收到的消息级别，每个级别都包含后面的级别。

#search_path = '"$user", public'	# schema names

#这个变量指定当一个对象(表、数据类型、函数等)被用一个无模式限定的简单名称引 用时，用于进行搜索该对象的模式顺序。当在不同模式中有同名
#对象时，将使用第一个在搜索路径中被找到的对象。一个不属于搜索路径中任何一个模式的对象只能通过用限定名(带点号)指定包含它的模式来引用。

#search_path的值必需是一个逗号分隔的模式名列表。任何不是一个已有模式的名称，或者是一个用户不具有USAGE权限的模式，将被安静地忽略。

#如果列表项之一是特殊名$user，
#则具有SESSION_USER返回的名字的模式将取代它(如果有这样一个模式并且该用户有该模式的USAGE权限;如果没有，$user会被忽略)。

#系统目录模式pg_catalog总是被搜索，不管它是否在搜索路径中被提及。如果它在路径中被提及，那么它将被按照路径指定的顺序搜索。
#如果pg_catalog不在路径中，则它将在任何路径项之前被搜索。

#同样，当前会话的临时表模式pg_temp_nnn也总是被搜索(如果存在)。它可以在路径中 通过使用别名pg_temp显式列出。如果在路径中没有列出，
#那么会首先对它进行搜索(甚 至是在pg_catalog之前)。
#然而，临时模式只被用来搜索关系(表、视图、序列等)和 数据类型名。它从不用于搜索函数或操作符名。

#当对象创建时没有指定一个特定目标模式，它们将被放置在search_path中第一个合法模 式中。如果搜索路径为空将报告一个错误。
#这个参数的缺省值是"$user", public。这种设置支持一个数据库(其中没有用户拥有私 有模式，并且所有人共享使用public)、每个用户私有模式#及其组合的共享使用。其它 效果可以通过全局或者针对每个用户修改默认搜索路径设置获得。
#搜索路径的当前有效值可以通过SQL函数current_schemas检查。它和检查search_path的值不太一样，
#因为current_schemas显示出现在search_path中的项如 何被解析。


#row_security = on

#这个变量控制是否以抛出一个错误来代替应用一条行安全性策略。在设置为on时，策略 正常应用。在设置为off时，只要有至少一条策略被应用则查询#就会失败。默认为on。受 限的行可见性会导致不正确的结果时，可将其改成off。例如，pg_dump默认会做这种更 改。这个变量对能绕过每一条行安#全性策略的角色(即超级用户和具有BYPASSRLS)属性 的角色没有效果。


#default_table_access_method = 'heap'

#如果 CREATE命令没有显式指定访问方法，则此参数指定在创建表或物化视图时使用的默认表访问方法，或者在使用SELECT ... INTO 时，不允许
#指定表访问方法。默认为heap。

#default_tablespace = ''		# a tablespace name, '' uses the default

#这个变量指定当一个CREATE命令没有显式指定一个表空间时，创建对象(表和索引)的 默认表空间。 它还决定了指导未来分区的分区关系的表空间。
#该值要么是一个表空间的名字，要么是一个指定使用当前数据库默认表空间的空字符 串。如果该值和任何现有表空间的名字都不匹配，PostgreSQL将#自动使用当前数据库的 默认表空间。如果指定了一个非默认的表空间，用户必须对它有CREATE权限，否则创建 企图将失败。
#这个变量不被用于临时表，对临时表会使用temp_tablespaces。
#当创建数据库时也会使用这个变量。默认情况下，一个新数据库会从它的模板数据库继 承其表空间设置。


#default_toast_compression = 'pglz'	# 'pglz' or 'lz4'

#toast 默认压缩方式

#temp_tablespaces = ''			# a list of tablespace names, '' uses
														# only default tablespace
														
#这个变量指定当一个CREATE命令没有显式指定一个表空间时，创建临时对象(临时表和 临时表上的索引)的默认表空间。用于排序大型数据集的临时文#件也被创建在这些表空 间中。
#该值是一个表空间名字的列表。当列表中有多于一个名称时，每次一个临时对象被创建 时PostgreSQL随机选择列表中的一个成员。例外是在一个事务#中，连续创建的临时对象 被放置在里表中的连续表空间中。如果列表被选中元素是一个空字符串，PostgreSQL将自动使用当前数据库的默认表空间。
#当temp_tablespaces被交互式地设置时，指定一个不存在的表空间是一种错误，类似于 为用户指定一个不具有CREATE权限的表空间。不过，当使
#用一个之前设置的值时，不存 在的表空间会被忽略，就像用户缺少CREATE权限的表空间一样。特殊地，使用一个 在postgresql.conf中设置的值
#时，这条规则起效。默认值是一个空字符串，它使得所有临时对象被创建在当前数据库的默认表空间中。参阅default_tablespace。

#check_function_bodies = on

#这个参数通常为打开 。当设置为off时，它禁用CREATE FUNCTION期间对函数体字符串的 验证。
#禁用验证避免了验证处理的副作用并且避免了如向前引用导致的伪肯定。在代表 其他用户载入函数之前设置这个参数为off;pg_dump会自动这样做。

#default_transaction_isolation = 'read committed'

#每个SQL事务都有一个隔离级别，可以是“读未提交”、“读已提交”、“可重复读”或者“可序列化”。
#这个参数控制每个新事务的默认隔离级别。默认是“读已提 交”。

#default_transaction_read_only = off

#一个只读的SQL事务不能修改非临时表。这个参数控制每个新事务的默认只读状态 。 默认是off(读/写)。

#default_transaction_deferrable = off

#当运行在可序列化隔离级别时，一个可延迟只读 SQL 事务可以在它被允许继续之前延迟 一段时间。但是，一旦它开始执行就不会产生任何用来保证可#序列化性的负荷;因此序 列化代码将没有任何理由因为并发更新而强制它中断，使得这个选项适合于长时间运行 的只读事务。
#这个参数控制每个新事务的默认可延迟状态。目前它对读写事务或者那些操作在低于可 序列化隔离级别上的事务无效。默认值是off。

#session_replication_role = 'origin'

#为当前会话控制复制相关的触发器和规则的触发。需要超级用户权限才能设置这个变 量，并且会导致丢弃任何之前缓存下来的查询计划。可能的值有#origin(默 认)、replica和local。
#这个设置的预期用途是由逻辑复制系统在应用所复制的更改时将它设置为replica。其效 果将是触发器和规则(没有对其默认配置做修改)在复制机上
#将不会被触发。更多信息 请参考ALTER TABLE的子句ENABLE TRIGGER以及ENABLE RULE。
#PostgreSQL在内部会把设置origin和local同样对待。第三方复制系统可能会把这两个值 用于其内部目的，例如把local用来标出一个不应复制其#更改的会话。
#因为外键被实现为触发器，将这个参数设置为replica还会禁用所有的外键检查，如果使 用不当可能会让数据处于一种不一致的状态。

#statement_timeout = 0			# in milliseconds, 0 is disabled

#中止任何使用了超过指定时间量的语句。 如果log_min_error_statement被设置 为ERROR或更低，语句如果超时也会被记录。 如果指定值时没有#单位，则以毫秒为单 位。一个零值(默认)将禁用超时。
#超时从命令到达服务器的时间开始计算，一直到它被服务器完成为止。 在扩展查询协议 中，当任何与查询相关的消息(解析、绑定、执行、描述)到达#时，超时就开始运行，并 且可以通过执行完成或同步消息来取消超时。
#我们不推荐在postgresql.conf中设置statement_timeout，因为它会影响所有会话。

#lock_timeout = 0			# in milliseconds, 0 is disabled

#如果任何语句在试图获取表、索引、行或其他数据库对象上的锁时等到超过指定的时间 量，该语句将被中止。 该时间限制独立地应用于每一次锁获取#尝试。该限制会应用到显 式锁定请求(如LOCK TABLE或不带NOWAIT的SELECT FOR UPDATE)和隐式获得的锁。 如 果指定值时没有单位，则以毫
#秒为单位。一个零值(默认)将禁用超时。
#与statement_timeout不同，这个超时只在等待锁时发生。注意如 果statement_timeout为非零，设置lock_timeout为相同或更大的值没有意#义，因为事务 超时将总是第一个被触发。 如果log_min_error_statement 被设置为ERROR 或更低，超 时的语句将被记录。
#我们不推荐在postgresql.conf中设置lock_timeout，因为它会影响所有会话。

#idle_in_transaction_session_timeout = 0	# in milliseconds, 0 is disabled

#终止任何已经闲置超过这个参数所指定时间量的打开事务的会话。这使得该会话所持有 的任何锁被释放，并且其所持有的连接槽可以被重用，
#它也允许只对这个事务可见的元 组被清理。
#如果指定值时没有单位，则以毫秒为单位。0值(默认值)禁用超时。

#idle_session_timeout = 0		# in milliseconds, 0 is disabled
#vacuum_freeze_table_age = 150000000

#当表的pg_class.relfrozenxid域达到该设置指定的年龄时，VACUUM会执行一次激进的扫 描。激进的扫描与常规VACUUM的不同在于它会访问每一#个可能包含未冻结 XID 或 者 MXID 的页面，而不只是那些可能包含死亡元组的页面。默认值 是 1.5 亿个事务。尽管用户可以把这个值设置为从 #0 到 20 亿，VACUUM会悄悄地将有效值设置为autovacuum_freeze_max_age值的95%，因此在表 上启动一次反回卷自动清理之前有机会进行一次#定期手动VACUUM。

#vacuum_freeze_min_age = 50000000

#指定VACUUM在扫描表时用来决定是否冻结行版本的切断年龄(以事务计)。默认值是 5 千万个事务。尽管用户可以将这个值设置为从 0 到 10 亿，#VACUUM会悄悄地将有效值设 置为autovacuum_freeze_max_age值的一半，这样在强制执行的自动清理之间不会有过短 的时间间隔。 

#vacuum_failsafe_age = 1600000000
#vacuum_multixact_freeze_table_age = 150000000

#如果表的pg_class.relminmxid域超过了这个设置指定的年龄，VACUUM会执行一次激进的 扫描。激进的扫描与常规VACUUM的区别在于它会访问每#一个可能包含未冻结 XID 或 者 MXID 的页面，而不是只扫描那些可能包含死亡元组的页面。默认值 是 1.5 亿个组合事务。尽管用户可以把这个
#值设置为从 0 到 20 亿，VACUUM会悄悄地将有效值设置为autovacuum_multixact_freeze_max_age值的 95%，因此在表上启动一次反回卷自动#清理之前有机会进行一次定期手动VACUUM。

#vacuum_multixact_freeze_min_age = 5000000

#指定VACUUM在扫描表时用来决定是否把组合事务 ID 替换为一个更新的事 务 ID 或组合事务 ID 的切断年龄(以组合事务计)。默认值 是 5 千万
#个组合事务。尽管用户可以将这个值设置为从 0 到 10 亿，VACUUM会悄悄地将有效值设置为autovacuum_multixact_freeze_max_age值的一 #半，这样在强制执行的自动清理之间不会有过短的时间间隔。

#vacuum_multixact_failsafe_age = 1600000000
#bytea_output = 'hex'			# hex, escape

#设置bytea类型值的输出格式。有效值是hex(默认)和 escape(传统的 PostgreSQL 格 式)。
#不管这个设置的值如何，bytea类型总是接受这两种格式的输 入。

#xmlbinary = 'base64'

#设置二进制值如何被编码为 XML。例如，这适用于通过xmlelement函数或xmlforest函数 将bytea值转换到 XML 值。
#可能的值有base64和hex，它们都是用 XML 模式标准定义的。 默认值是base64。
#这里的实际选择都是根据爱好做出的，只受客户端应用中可能存在的限制的约束。两种方法都支持所有可能的值,尽管十六进制将比base64编码更大。

#xmloption = 'content'

#当在XML和字符串值之间进行转换时，无论设置DOCUMENT或 CONTENT都是隐式的。 有效值是DOCUMENT和CONTENT。 缺省值是CONTENT。
#当在 XML 和字 符串值之间进行转换时，设置DOCUMENT或CONTENT都是隐式的。 
#根据 SQL 标准，设置这个选项的命令是:
#             SET XML OPTION { DOCUMENT | CONTENT };
#这种语法在 PostgreSQL 也可用。

#gin_pending_list_limit = 4MB

#设置fastupdate被启用时可以使用的 GIN索引的待处理列表的最大尺寸。 如果 该列表增长到超过这个最大尺寸，会通过批量将其中的项移入索引的#主 GIN 数 据结构来清理列表。 如果指定值时没有单位，则以千字节为单位。默认值是四兆字节 (4MB)。 可以通过更改索引的存储参数来为个别 #GIN 索引覆盖这个设置。
```

#### 2.10.2 区域和格式化

```
# - Locale and Formatting -

datestyle = 'iso, ymd'

#设置日期和时间值的显示格式，以及解释有歧义的日期输入值的规则。
#由于历史原因， 这个变量包含两个独立的部分:输出格式声明(ISO、Postgres、SQL或German)、输入/输出的年/月/日顺序(DMY、MDY或YMD)。
#这些可以被独立设置或者一起设置。关键字Euro和European是DMY的同义词;关键字US、NonEuro和NonEuropean是MDY的同义词。
#内建默认值是ISO, MDY，但是initdb将用对应于选中的lc_time区域行 为的设置初始化配置文件。

#intervalstyle = 'postgres'

#设置间隔值的显示格式。

timezone = 'PRC'

#设置用于显示和解释时间戳的时区。

#timezone_abbreviations = 'Default'     # Select the set of available time zone
                                        # abbreviations.  Currently, there are
                                        #   Default
                                        #   Australia (historical usage)
                                        #   India
                                        # You can create your own file in
                                        # share/timezonesets/.
#设置服务器接受的日期时间输入中使用的时区缩写集合。默认值为'Default'， 这个集 合在全世界大多数地方都能工作。
#也还#】有'Australia'和'India'，以及可能为一种特定 安装定义的其他集合。		

#extra_float_digits = 1			# min -15, max 3; any value >0 actually
														# selects precise output mode
														
#这个参数调整用于文本输出浮点值的位数，包括float4，float8以及几何数据类型。

#client_encoding = sql_ascii		# actually, defaults to database
																# encoding
#设置客户端编码(字符集)

# These settings are initialized by initdb, but they can be changed.
lc_messages = 'en_US.UTF-8'			# locale for system error message
																# strings
#设置消息显示的语言。		

lc_monetary = 'zh_CN.UTF-8'			# locale for monetary formatting

#设置用于格式化货币量的区域

lc_numeric = 'zh_CN.UTF-8'			# locale for number formatting

#设置用于格式化数字的区域

lc_time = 'zh_CN.UTF-8'				# locale for time formatting

#设置用于格式化日期和时间的区域

# default configuration for text search
default_text_search_config = 'pg_catalog.english'

#选择被那些没有显式参数指定配置的文本搜索函数变体使用的文本搜索配置。
```

#### 2.10.3 共享库预载入

```
# - Shared Library Preloading -

#local_preload_libraries = ''

#这个变量指定一个或者多个要在连接开始时预载入的共享库。它包含一个由逗号分隔的 库名列表，其中每个名称都会按LOAD命令的方式解析。项之间
#的空格会被忽略，如果需 要在库名中包含空格或者逗号，请把库名放在双引号内。这个参数 值只在连接开始时生 效。后续的更改不会有任何效果。
#如果一个指定的 库没有找到，连接尝试将会失败。
#任何用户都能设置这个选项。正因为如此，能被这样载入的库被严格限制为出现于安装 的标准库 目录中plugins子目录下的共享库(保证只有“安全
#的”库被安装到 这里是 数据库管理员的责任)。local_preload_libraries中的项可以显式 指定这个目录，例 如$libdir/plugins/mylib，
#或者只是指定库的 名称 — mylib 和 $libdir/plugins/ mylib的效果是相同的。
#这个特性的目的是允许非特权用户在特定的会话中载入正在调试的或者性能度量库， 而 无需一个显式的LOAD命令。为了这个目的，通常通过使用客 #户端的PGOPTIONS环境变量 或者 ALTER ROLE SET来设置这个参数。
#不过，除非一个模块被特别设计成由非超级用户以这种方式使用，通常不推荐使用这个 设置。应该看看 session_preload_libraries。

#session_preload_libraries = ''

#这个变量指定一个或者多个要在连接开始时预载入的共享库。只有超级用户更够更改这 个设置。它包含一个由逗号分隔的库名列表，其中每个名称都会#按LOAD命令的方式解 析。项之间的空格会被忽略，如果需要在库名中包含空格或者逗号，请把库名放在双引 号内。这个参数只在连接开始时起效。后#续的改变没有效果。如果指定的库没有找到， 连接尝试将会失败。只有超级用户能够更改这个设置。
#这个特性的意图是允许在特定会话中载入调试用的或者测量性能的库，而不需要显式的 给出一个 LOAD命令。例如，通过用ALTER ROLE SET设置这
#个参数可以 为一个给定用户 名下的所有会话启用auto_explain。还有，无需重启 服务器就能更改这个参数(但是只 有新会话启动时才会生效)，
#这样可以以这种方式更容易地增 加新模块，即便它们会应 用到所有会话。
#和shared_preload_libraries不同，相对于在库被第一次使用 时载入它，在会话开始时 载入库并没有什么性能优势。不过，当使用连接池时这样#做还是有一些 优势。


#shared_preload_libraries = ''	# (change requires restart)

#这个变量指定一个或者多个要在服务器启动时预载入的共享库。它包含一个由逗号分隔 的库名列表，其中每个名称都会按LOAD命令的方式解析。项之
#间的空格会被忽略，如果 需要在库名中包含空格或者逗号，请把库名放在双引号内。这个参数只能在服务器启动 时设置。如果指定的库没有找到，服#务器将无法启动。
#有些库需要执行只能在postmaster启动时发生的特定操作，例如分配共享内存、保留轻 量级锁 或者启动后台工作者。这些库必须通过这个参数在服#务器启动时载入。每个库的 详情请见文档。
#其他库也能被预载入。通过预载入一个共享库，当该库被第一次使用时就可以避免库的 启动时间。 不过，启动每个新服务器进程的时间可能会略有增#加，即使该进程从不使 用该库。因此，推荐只 把这个参数用于那些要在大多数会话中使用的库上。还有，改变 这个参数要求重启服务器，因此 对于#短期的调试任务来说这不是好的选择，应该转用 session_preload_libraries。

#jit_provider = 'llvmjit'		# JIT library to use

#这个变量是要被使用的JIT提供者库的名称。默认是llvmjit。这个 参数只能在服务器启动时设置。
#如果这个变量被设置为一个不存在的库，JIT将不可用，但是也不会发生错误。这种特性 允许在主PostgreSQL包之外单独安装JIT支持。

```

#### 2.10.4 其他默认值

```
# - Other Defaults -

#dynamic_library_path = '$libdir'

#如果需要打开一个可以动态装载的模块并且在CREATE FUNCTION或LOAD命令中指定的文件 名没有目录部分(即名字中不包含斜线)，那么系统将搜索#这个路径以查找所需的文 件。

#dynamic_library_path的值必须是一个冒号分隔(或者在 Windows上以分号分隔)的绝 对目录路径的列表。如果一个列表元素以特殊字符串开始，#$libdir会被替换为PostgreSQL包中已编译好的库目录。
#这里是PostgreSQL发布提供的模块被安装的位置 (使用pg_config --pkglibdir来找到这个目录的名字)。
#例如:
#dynamic_library_path = '/usr/local/lib/postgresql:/home/my_project/lib: $libdir'
#或者在 Windows 环境中:
#dynamic_library_path = 'C:\tools\postgresql;H:\my_project\lib;$libdir'
#这个参数的默认值是'$libdir'。如果该值被设置为一个空字符串，则关闭自动路径搜 索。
#这个参数可以在运行时由超级用户修改，但是这样修改的设置只能保持到这个客户端连 接的结尾，因此这个方法应该保留给开发目的。 
#我们建议在postgresql.conf配置文件 中设置这个参数。

#gin_fuzzy_search_limit = 0

#GIN 索引返回的集合尺寸的软上限。
```

### 2.11 锁管理

```
#------------------------------------------------------------------------------
# LOCK MANAGEMENT
#------------------------------------------------------------------------------

#deadlock_timeout = 1s

#这是进行死锁检测之前在一个锁上等待的时间量。死锁检测相对昂贵，因此服务器不会 在每次等待锁时都运行这个它。 我们乐观地假设在生产应用中#死锁是不常出现的，并且 只在开始检测死锁之前等待一会儿。增加这个值就减少了浪费在无用的死锁检测上的时 间，但是减慢了报告真正死锁错误的#速度。 如果指定值时没有单位，则以毫秒为单位。 默认是 1 秒(1s)，这可能是实际中你想要的最小值。 在一个高负载的服务器上，你可 能需要增#大它。这个值的理想设置应该超过你通常的事务时间，这样就可以减少在锁释 放之前就开始死锁检查的机会。只有超级用户可以更改这个设置。
#当log_lock_waits被设置时，这个参数还可以决定发出关于锁等待的日志之前等待的时 间量。如果你想调查锁延迟，你可能希望设置一个比正常的#deadlock_timeout小的值。

#max_locks_per_transaction = 64		# min 10
					                        # (change requires restart)
#共享锁表跟踪在max_locks_per_transaction * (max_connections + max_prepared_transactions) 个对象(如表)上的锁。因此，在任#何一个时刻，只有不 超过这么多个可区分对象能够被锁住。这个参数控制为每个事务分配的对象锁的平均数 量。个体事务可以锁住更多对象，数量可#以和锁表中能容纳的所有事务的锁一样多。 这不是能被锁住的行数，那个值是没有限制的。默认值 64 已经被历史证明是足够的， 但是如果你有需要#在一个事务中使用很多不同表的查询(例如查询一个有很多子表的父 表)，你可能需要提高这个值。这个参数只能在服务器启动时设置。
#当运行一个后备服务器时，你必须设置这个参数为大于等于主服务器上的值。否则，后 备服务器上将不允许查询。					
					
#max_pred_locks_per_transaction = 64	# min 10
					# (change requires restart)

#共享谓词锁表跟踪在max_pred_locks_per_transaction * (max_connections + max_prepared_transactions) 个对象(如表)上的锁。#因此，在任何一个时刻，只有不 超过这么多个可区分对象能够被锁住。这个参数控制为每个事务分配的对象锁的平均数 量。个体事务可以锁住更多对#象，数量可以和锁表中能容纳的所有事务的锁一样多。 这不是能被锁住的行数，那个值是没有限制的。默认值 64 已经在测试中被证明通常是 足够#的，但是如果你有需要在一个可序列化事务中使用很多不同表的查询(例如查询一 个有很多子表的父表)，你可能需要提高这个值。这个参数只能在服务#器启动时设置。
					
#max_pred_locks_per_relation = -2	# negative values mean
                                  # (max_pred_locks_per_transaction
                                  #  / -max_pred_locks_per_relation) - 1
					
#这个参数控制在谓词锁被提升为覆盖整个关系之前，该谓词锁能够在单个关系上锁住多 少页面或元组。大于等于零的值表示一种绝对限制，而负值表示 #用max_pred_locks_per_transaction除以这个设置的绝对值。默认值为-2，它将保持以往版本的PostgreSQL中的行为。这个参数只能在#postgresql.conf文件中或者服务器命令 行上设置。

#max_pred_locks_per_page = 2            # min 0

#这个参数控制在谓词锁被提升为覆盖整个页面之前，该谓词锁能在单一页面上锁住多少 行。默认值是2。这个参数只能在postgresql.conf文件中或#者服务器命令行上设置。
```

### 2.12 版本和平台兼容性

#### 2.12.1 历史版本

```
#------------------------------------------------------------------------------
# VERSION AND PLATFORM COMPATIBILITY
#------------------------------------------------------------------------------

# - Previous PostgreSQL Versions -

#array_nulls = on

#这个参数控制数组输入解析器是否把未用引号的NULL识别为一个空数组元素。默认 为on，允许输入包含空值的数组值。但是PostgreSQL 8.2 之前#的版本不支持数组中的空 值，并且因此将把NULL当作指定一个值为字符串“NULL”的正常数组元素。对于那些要 求旧行为的应用的向后兼容性，这个#变量可以被设置为off。
#注意即使这个变量为off也能够创建包含空值的数组值。

#backslash_quote = safe_encoding	# on, off, or safe_encoding

#这个参数控制字符串文本中的单引号是否能够用\'来表示。首选的 SQL 标准的方法是将 其双写('')，但是PostgreSQL在历史上也接受\'。不过使#用\'容易导致安全风险，因 为在某些客户端字符集编码中，有多字节字符的最后一个字节在数值上等价于 ASCII 的\'。如果客户端代码没有做到正#确转义，那么将会导致 SQL 注入攻击。如果服务器拒 绝看起来带有被反斜线转义的单引号的查询，那么就可以避免这种风 险。
#backslash_quote的可用值是on(总是允许\')、off(总是拒绝)以 及safe_encoding(只有客户端编码不允许在多字节字符中存在 ASCII \时允 #许)。safe_encoding是默认设置。
#注意在符合标准的字符串文本中，\就表示\。这个参数只影响不符合标准的文本的处 理，包括转义字符串语法(E'...')。

#escape_string_warning = on

#打开时，如果在普通字符串文本中('...'语法)出现了 一个反斜线(\)并 且standard_conforming_strings为关闭，那么就会发出一个警告。默#认值是on。
#希望使用反斜线作为转义符的应用应该被修改来使用转义字符串语法(E'...')，因为 在 SQL 标准中普通字符串的默认行为是将反斜线视作一个普通#字符。这个变量可以被启 用来帮助定位需要被更改的代码。

#lo_compat_privileges = off

#在PostgreSQL 9.0 之前，大对象不具有访问特权并且因此总是所有用户可读可写的。为 了和以前的版本兼容，把这个变量设置为on可以禁用这种
#新的特权检查。默认是off。只 有超级用户可以更改这个设置。
#设置这个参数不会禁用所有与大对象相关的安全检查 — 除了那些在PostgreSQL 9.0中 已经修改了的默认行为。


#quote_all_identifiers = off

#当数据库产生 SQL 时，强制所有标识符被引号包围，即使它们(当前)不是关键字。这 将影响EXPLAIN的输出以及pg_get_viewdef等函数的结果。#另请参 阅pg_dump和pg_dumpall的--quote-all-identifiers选项。

#standard_conforming_strings = on

#控制普通字符串文本('...')是否按照 SQL 标准把反斜线当普通文本。从PostgreSQL 9.1 开始，默认值为on(之前的发行中默认值为off)。应用#可以检查这个参数来判断字 符串文本如何被处理。这个参数的存在也可以被当做转义字符串语法(E'...')被支持 的标志。如果一个应用希望反斜线
#被当做转义字符，应该使用转义字符串语法。

#synchronize_seqscans = on


#它允许对大型表的顺序扫描与其他扫描同步，因此并发扫描可以在几乎相同的时刻读取 相同的块，这样可以分担 I/O 负载。当启用这个参数时，一个#扫描可能会从表的中间开 始并且之后“绕回”到开头以覆盖所有的行，这样可以与已在进行中的扫描活动同步。 对于没有ORDER BY子句的查询来，这#样的扫描会在返回行的顺序中造成不可预料的改 变。将这个参数设置为off以保证 8.3之前的行为(顺序扫描总是从表的起始处开始)。默认值是on。
```

#### 2.12.2 平台和客户端兼容性

```
# - Other Platforms and Clients -

#transform_null_equals = off

#当打开时，形为expr = NULL(或NULL = expr)的表达式将被当做expr IS NULL， 也就是 说，如果expr得出空值值则返回真，否则返回假。正#确的 SQL 标准兼容的expr = NULL行 为总是返回空(未知)。因此这个参数默认为off。
#不过，在Microsoft Access里的过滤表单生成的查询似乎使用expr = NULL来测试空值， 因此，如果你使用这个接口访问数据库，你可能想把这
#个选项打开。因为expr = NULL形 式的表达式总是返回空值(使用 SQL 标准解释)。它们不是非常有用并且在普通应用中 也不常见，在应用中也不常#见，因此这个选项实际上没有什么危害。但是新用户常常对 涉及 空值的表达式的语义上感到困惑，因此这个选项默认为关闭。
#请注意这个选项只影响= NULL形式，而不影响其它比较操作符或者其它与一些涉及等值 操作符的表达式在计算上等效的其他表达式(例如IN)。因此，#这个选项不是劣质程序 的一般修复。
```

### 2.13 错误处理

```
#------------------------------------------------------------------------------
# ERROR HANDLING
#------------------------------------------------------------------------------

#exit_on_error = off			# terminate session on any error?

#如果为开(on)，任何错误将中止当前会话。默认情况下，这个值被设置为关(off)，这样 只有 FATAL 错误(致命)将中止会话。

#restart_after_crash = on		# reinitialize after backend crash?

#当被设置为开(on)，PostgreSQL将在一次后端崩溃后自动重新初始化。 让这个值设置为 开(on)通常是将数据库可用性最大化的最佳方法。
#但是在某些环境中，例如PostgreSQL 被集群软件调用时，禁用重启可能很有用，这样集群软件可以得到控制并且采取它认为 适当的行动。

#data_sync_retry = off			# retry or panic on failure to fsync
                            # data?
                            # (change requires restart)
                            
#如果设置为关(off)，PostgreSQL在将修改的数据文件刷新到文件系统失败时，将引发 PANIC级错误。 这样会导致数据库服务器崩溃。这个参数只#能在服务器启动时被设置。
#在某些操作系统上，回写失败后，内核页面缓存中的数据状态未知。 在某些情况下，它 可能已被完全遗忘，因此重试不安全;第二次尝试可能报告为
#成功，而事实上数据已丢 失。在此类情形下，避免数据丢失的唯一方法是在报告任何故障后从WAL中恢复，最好是 在调查了故障的根本原因并更换了任#何有故障的硬件之后。
#如果设置为开(on)，PostgreSQL将报告错误，但会继续运行，以便可以在以后的检查点 中重试数据刷新操作。 仅在调查操作系统假如回写失败时对#缓冲数据的处理方式的情况 下，才将其设置为开(on)。

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

