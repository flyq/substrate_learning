# 源码分析一
之前也没有做过类似的源码分析，而且我也是近段时间才开始看 `Rust` ，有些代码还需要 `Google` ，所以这里就算是记录一下我的思考吧。

之前看其他的源码分析时，都是按照模块来介绍，整体介绍几处就完事，总觉得少了什么。我想尝试根据函数的运行调用来。

## 进程的入口
在  [ `node` 文件夹下的 `main` 函数中](https://github.com/paritytech/substrate/blob/master/node/src/main.rs#L46-L61)：
```rust
fn main() {
	let version = VersionInfo {
		name: "Substrate Node",
		commit: env!("VERGEN_SHA_SHORT"),
		version: env!("CARGO_PKG_VERSION"),
		executable_name: "substrate",
		author: "Parity Technologies <admin@parity.io>",
		description: "Generic substrate node",
		support_url: "https://github.com/paritytech/substrate/issues/new",
	};

	if let Err(e) = cli::run(::std::env::args(), Exit, version) {
		eprintln!("Error starting the node: {}\n\n{:?}", e, e);
		std::process::exit(1)
	}
}
```
可以看出调用 `cli` 模块里面的 `run` 函数，并且传入了标准库里面的 `args()` 返回的结果，两个其他结构体类型的变量作为参数。标准库里面的 `args()` 的功能是得到命令行里面输入的参数，详见[链接](https://doc.rust-lang.org/std/env/fn.args.html)。

然后看一下这个 `run` 函数，在 [node/cli/src/lib.rs](https://github.com/paritytech/substrate/blob/master/node/cli/src/lib.rs#L143-L201) ：
```rust 
/// Parse command line arguments into service configuration.
pub fn run<I, T, E>(args: I, exit: E, version: cli::VersionInfo) -> error::Result<()> where
	I: IntoIterator<Item = T>,
	T: Into<std::ffi::OsString> + Clone,
	E: IntoExit,
{
	let ret = cli::parse_and_execute::<service::Factory, CustomSubcommands, NoCustom, _, _, _, _, _>(
		load_spec, &version, "substrate-node", args, exit,
		|exit, _cli_args, _custom_args, config| {
			info!("{}", version.name);
			info!("  version {}", config.full_version());
			info!("  by Parity Technologies, 2017-2019");
			info!("Chain specification: {}", config.chain_spec.name());
			info!("Node name: {}", config.name);
			info!("Roles: {:?}", config.roles);
			let runtime = RuntimeBuilder::new().name_prefix("main-tokio-").build()
				.map_err(|e| format!("{:?}", e))?;
			match config.roles {
				ServiceRoles::LIGHT => run_until_exit(
					runtime,
					service::Factory::new_light(config).map_err(|e| format!("{:?}", e))?,
					exit
				),
				_ => run_until_exit(
					runtime,
					service::Factory::new_full(config).map_err(|e| format!("{:?}", e))?,
					exit
				),
			}.map_err(|e| format!("{:?}", e))
		}
	);

	match &ret {
		Ok(Some(CustomSubcommands::Factory(cli_args))) => {
			let config = cli::create_config_with_db_path::<service::Factory, _>(
				load_spec,
				&cli_args.shared_params,
				&version,
			)?;

			match ChainSpec::from(config.chain_spec.id()) {
				Some(ref c) if c == &ChainSpec::Development || c == &ChainSpec::LocalTestnet => {},
				_ => panic!("Factory is only supported for development and local testnet."),
			}

			let factory_state = FactoryState::new(
				cli_args.mode.clone(),
				cli_args.num,
				cli_args.rounds,
			);
			transaction_factory::factory::<service::Factory, FactoryState<_>>(
				factory_state,
				config,
			).map_err(|e| format!("Error in transaction factory: {}", e))?;

			Ok(())
		},
		_ => ret.map_err(Into::into).map(|_| ())
	}
}
```
这是一个参数为泛型的函数，输入三个变量名分别为 `args` ， `exit` ， `version` 的变量，其中 `args` 所属的类型约束为可迭代的类型，并且它还需要是可以转换成 [OsString](https://doc.rust-lang.org/std/ffi/struct.OsString.html) ，它还需可以是 `Clone` 的， `Rust` 里面，像 `String` 这种保存在 `heap` 里面的数据都是可以 `Clone` 的，详见[链接](https://rustlang-cn.org/users/book-exp/04%20Understanding%20Ownership/01%20What%20is%20Ownership.html#%E5%8F%98%E9%87%8F%E5%92%8C%E6%95%B0%E6%8D%AE%E7%9A%84%E4%BA%A4%E4%BA%92%EF%BC%9Aclone)。 `exit` 所属的类型约束到 `IntoExit trait` ，`IntoExit` 具体下面会讲。 `version` 是 自定义 `struct` ，参见：
```rust
/// Executable version. Used to pass version information from the root crate.
pub struct VersionInfo {
	/// Implemtation name.
	pub name: &'static str,
	/// Implementation version.
	pub version: &'static str,
	/// SCM Commit hash.
	pub commit: &'static str,
	/// Executable file name.
	pub executable_name: &'static str,
	/// Executable file description.
	pub description: &'static str,
	/// Executable file author.
	pub author: &'static str,
	/// Support URL.
	pub support_url: &'static str,
}
```
 `trait` 在 `Rust` 里面是描写某些类型/自定义 `struct` 的行为（有什么共同方法）的数据结构。 `IntoExit trait` 是一个描述那些可以被转换成退出信号的信号，参考：
 ```rust
 /// https://github.com/paritytech/substrate/blob/master/core/cli/src/lib.rs#L95-L101

 /// Something that can be converted into an exit signal.
pub trait IntoExit {
	/// Exit signal type.
	type Exit: Future<Item=(),Error=()> + Send + 'static;
	/// Convert into exit signal.
	fn into_exit(self) -> Self::Exit;
}


// https://github.com/paritytech/substrate/blob/master/node/src/main.rs#L27-L44

// handles ctrl-c
pub struct Exit;
impl IntoExit for Exit {
	type Exit = future::MapErr<oneshot::Receiver<()>, fn(oneshot::Canceled) -> ()>;
	fn into_exit(self) -> Self::Exit {
		// can't use signal directly here because CtrlC takes only `Fn`.
		let (exit_send, exit) = oneshot::channel();

		let exit_send_cell = RefCell::new(Some(exit_send));
		ctrlc::set_handler(move || {
			if let Some(exit_send) = exit_send_cell.try_borrow_mut().expect("signal handler not reentrant; qed").take() {
				exit_send.send(()).expect("Error sending exit notification");
			}
		}).expect("Error setting Ctrl-C handler");

		exit.map_err(drop)
	}
}
```
接下来调用 `cli::parse_and_execute` ，这个是根据接收到的命令行参数进行不同分支执行的逻辑。


参考[链接](https://github.com/paritytech/substrate/blob/master/core/cli/src/lib.rs#L169-L239)：
```rust
/// Parse command line interface arguments and executes the desired command.
///
/// # Return value
///
/// A result that indicates if any error occurred.
/// If no error occurred and a custom subcommand was found, the subcommand is returned.
/// The user needs to handle this subcommand on its own.
///
/// # Remarks
///
/// `CC` is a custom subcommand. This needs to be an `enum`! If no custom subcommand is required,
/// `NoCustom` can be used as type here.
/// `RP` are custom parameters for the run command. This needs to be a `struct`! The custom
/// parameters are visible to the user as if they were normal run command parameters. If no custom
/// parameters are required, `NoCustom` can be used as type here.
pub fn parse_and_execute<'a, F, CC, RP, S, RS, E, I, T>(
	spec_factory: S,
	version: &VersionInfo,
	impl_name: &'static str,
	args: I,
	exit: E,
	run_service: RS,
) -> error::Result<Option<CC>>
where
	F: ServiceFactory,
	S: FnOnce(&str) -> Result<Option<ChainSpec<FactoryGenesis<F>>>, String>,
	CC: StructOpt + Clone + GetLogFilter,
	RP: StructOpt + Clone + AugmentClap,
	E: IntoExit,
	RS: FnOnce(E, RunCmd, RP, FactoryFullConfiguration<F>) -> Result<(), String>,
	I: IntoIterator<Item = T>,
	T: Into<std::ffi::OsString> + Clone,
{
	panic_handler::set(version.support_url);

	let full_version = service::config::full_version_from_strs(
		version.version,
		version.commit
	);

	let matches = CoreParams::<CC, RP>::clap()
		.name(version.executable_name)
		.author(version.author)
		.about(version.description)
		.version(&(full_version + "\n")[..])
		.setting(AppSettings::GlobalVersion)
		.setting(AppSettings::ArgsNegateSubcommands)
		.setting(AppSettings::SubcommandsNegateReqs)
		.get_matches_from(args);
	let cli_args = CoreParams::<CC, RP>::from_clap(&matches);

	init_logger(cli_args.get_log_filter().as_ref().map(|v| v.as_ref()).unwrap_or(""));
	fdlimit::raise_fd_limit();

	match cli_args {
		params::CoreParams::Run(params) => run_node::<F, _, _, _, _>(
			params, spec_factory, exit, run_service, impl_name, version,
		).map(|_| None),
		params::CoreParams::BuildSpec(params) =>
			build_spec::<F, _>(params, spec_factory, version).map(|_| None),
		params::CoreParams::ExportBlocks(params) =>
			export_blocks::<F, _, _>(params, spec_factory, exit, version).map(|_| None),
		params::CoreParams::ImportBlocks(params) =>
			import_blocks::<F, _, _>(params, spec_factory, exit, version).map(|_| None),
		params::CoreParams::PurgeChain(params) =>
			purge_chain::<F, _>(params, spec_factory, version).map(|_| None),
		params::CoreParams::Revert(params) =>
			revert_chain::<F, _>(params, spec_factory, version).map(|_| None),
		params::CoreParams::Custom(params) => Ok(Some(params)),
	}
}
```
 `parse_and_execute` 函数接受一些参数，包括子命令 `CC` ，命令参数 `RP`，然后通过匹配枚举类型 `CoreParams`，选择不同命令执行分支。
 ```rust
/// https://github.com/paritytech/substrate/blob/master/core/cli/src/params.rs#L624-L740

 /// All core commands that are provided by default.
///
/// The core commands are split into multiple subcommands and `Run` is the default subcommand. From
/// the CLI user perspective, it is not visible that `Run` is a subcommand. So, all parameters of
/// `Run` are exported as main executable parameters.
#[derive(Debug, Clone)]
pub enum CoreParams<CC, RP> {
	/// Run a node.
	Run(MergeParameters<RunCmd, RP>),

	/// Build a spec.json file, outputing to stdout.
	BuildSpec(BuildSpecCmd),

	/// Export blocks to a file.
	ExportBlocks(ExportBlocksCmd),

	/// Import blocks from file.
	ImportBlocks(ImportBlocksCmd),

	/// Revert chain to the previous state.
	Revert(RevertCmd),

	/// Remove the whole chain data.
	PurgeChain(PurgeChainCmd),

	/// Further custom subcommands.
	Custom(CC),
}

impl<CC, RP> StructOpt for CoreParams<CC, RP> where
	CC: StructOpt + GetLogFilter,
	RP: StructOpt + AugmentClap
{
	fn clap<'a, 'b>() -> App<'a, 'b> {
		RP::augment_clap(
			RunCmd::augment_clap(
				CC::clap().unset_setting(AppSettings::SubcommandRequiredElseHelp)
			)
		).subcommand(
			BuildSpecCmd::augment_clap(SubCommand::with_name("build-spec"))
				.about("Build a spec.json file, outputing to stdout.")
		)
		.subcommand(
			ExportBlocksCmd::augment_clap(SubCommand::with_name("export-blocks"))
				.about("Export blocks to a file.")
		)
		.subcommand(
			ImportBlocksCmd::augment_clap(SubCommand::with_name("import-blocks"))
				.about("Import blocks from file.")
		)
		.subcommand(
			RevertCmd::augment_clap(SubCommand::with_name("revert"))
				.about("Revert chain to the previous state.")
		)
		.subcommand(
			PurgeChainCmd::augment_clap(SubCommand::with_name("purge-chain"))
				.about("Remove the whole chain data.")
		)
	}

	fn from_clap(matches: &::structopt::clap::ArgMatches) -> Self {
		match matches.subcommand() {
			("build-spec", Some(matches)) =>
				CoreParams::BuildSpec(BuildSpecCmd::from_clap(matches)),
			("export-blocks", Some(matches)) =>
				CoreParams::ExportBlocks(ExportBlocksCmd::from_clap(matches)),
			("import-blocks", Some(matches)) =>
				CoreParams::ImportBlocks(ImportBlocksCmd::from_clap(matches)),
			("revert", Some(matches)) => CoreParams::Revert(RevertCmd::from_clap(matches)),
			("purge-chain", Some(matches)) =>
				CoreParams::PurgeChain(PurgeChainCmd::from_clap(matches)),
			(_, None) => CoreParams::Run(MergeParameters::from_clap(matches)),
			_ => CoreParams::Custom(CC::from_clap(matches)),
		}
	}
}

impl<CC, RP> GetLogFilter for CoreParams<CC, RP> where CC: GetLogFilter {
	fn get_log_filter(&self) -> Option<String> {
		match self {
			CoreParams::Run(c) => c.left.get_log_filter(),
			CoreParams::BuildSpec(c) => c.get_log_filter(),
			CoreParams::ExportBlocks(c) => c.get_log_filter(),
			CoreParams::ImportBlocks(c) => c.get_log_filter(),
			CoreParams::PurgeChain(c) => c.get_log_filter(),
			CoreParams::Revert(c) => c.get_log_filter(),
			CoreParams::Custom(c) => c.get_log_filter(),
		}
	}
}
```
如果运行 `cargo run \-- --help` ，就能看到对应的子命令。
```bash
SUBCOMMANDS:
    build-spec       Build a spec.json file, outputing to stdout.
    export-blocks    Export blocks to a file.
    factory          Manufactures num transactions from Alice to random accounts. Only supported for development or
                     local testnet.
    help             Prints this message or the help of the given subcommand(s)
    import-blocks    Import blocks from file.
    purge-chain      Remove the whole chain data.
    revert           Revert chain to the previous state.
```
我们把精力放在 `Run(MergeParameters<RunCmd, RP>)`

