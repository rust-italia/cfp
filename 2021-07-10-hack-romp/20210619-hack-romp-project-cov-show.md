```
src/body_parser/fixed_length.rs:
    1|       |use crate::body_parser::{BodyParser, ParserState};
    2|       |use crate::message::stomp_message::StompMessage;
    3|       |
    4|       |/// STOMP protocol push parser, parses message body.
    5|       |///
    6|       |/// readonly bytes pushed in from the network and copied into the message
    7|       |///
    8|       |/// State of the Struct
    9|       |
   10|       |/// state of the loop
   11|     12|#[derive(Debug, PartialEq)]
   12|       |enum State {
   13|       |    Start,
   14|       |    /// expecting trailing \0
   15|       |    AlmostDone,
   16|       |    Done,
   17|       |    Error,
   18|       |}
   19|       |
   20|       |/// Body parser for binary data when we know the expected length of the data, content-len: x was provided.
   21|       |pub struct FixedLengthBodyParser {
   22|       |    state: State,
   23|       |    expected_len: usize,
   24|       |    remaining_len: usize,
   25|       |}
   26|       |
   27|       |/// reads up to content-length: xxx bytes
   28|       |impl FixedLengthBodyParser {
   29|      1|    pub fn new() -> FixedLengthBodyParser {
   30|      1|        FixedLengthBodyParser {
   31|      1|            state: State::Start,
   32|      1|            expected_len: 0,
   33|      1|            remaining_len: 0,
   34|      1|        }
   35|      1|    }
   36|       |
   37|       |    /// must be called before use
   38|      2|    pub fn reinit(&mut self, expected_len: usize) {
   39|      2|        self.state = State::Start;
   40|      2|        self.remaining_len = expected_len;
   41|      2|        self.expected_len = expected_len;
   42|      2|    }
   43|       |
   44|      0|    pub fn expected_len(&self) -> usize {
   45|      0|        self.expected_len
   46|      0|    }
   47|       |}
   48|       |
   49|       |impl BodyParser for FixedLengthBodyParser {
   50|      6|    fn push(&mut self, buffer: &[u8], message: &mut StompMessage) -> Result<usize, ParserState> {
   51|      6|        let len = buffer.len();
   52|      6|
   53|      6|        // TODO might not be necessary if ([0..0]) is sane
   54|      6|        if State::AlmostDone == self.state {
   55|      0|            let zero = buffer[0];
   56|      0|            if zero != b'\0' {
   57|      0|                self.state = State::Error;
   58|      0|                return Err(ParserState::InvalidMessage);
   59|       |            } else {
   60|      0|                self.state = State::Done;
   61|      0|                return Ok(1);
   62|       |            }
   63|      6|        }
   64|      6|        if len < self.remaining_len {
   65|      4|            message.add_body(buffer);
   66|      4|            self.remaining_len -= len;
   67|      4|
   68|      4|            return Ok(len);
   69|      2|        } else if len == self.remaining_len {
   70|      0|            message.add_body(buffer);
   71|      0|            self.remaining_len -= len;
   72|      0|            self.state = State::AlmostDone;
   73|      0|
   74|      0|            return Ok(len);
   75|      2|        } else if len > self.remaining_len {
   76|      2|            let to_read = self.remaining_len;
   77|      2|            message.add_body(&buffer[0..to_read]);
   78|      2|            self.remaining_len -= to_read;
   79|      2|
   80|      2|            // verify trailing \0 is present
   81|      2|            match buffer[to_read] {
   82|       |                b'\0' => {
   83|      1|                    self.state = State::Done;
   84|      1|                    return Ok(to_read + 1);
   85|       |                }
   86|       |                _ => {
   87|      1|                    self.state = State::Error;
   88|      1|                    return Err(ParserState::InvalidMessage);
   89|       |                }
   90|       |            }
   91|      0|        }
   92|      0|
   93|      0|        // unreachable
   94|      0|        return Err(ParserState::InvalidMessage);
   95|      6|    }
   96|       |
   97|      6|    fn is_done(&self) -> bool {
   98|      6|        State::Done == self.state
   99|      6|    }
  100|       |}
  101|       |
  102|       |#[cfg(test)]
  103|       |mod tests {
  104|       |    use crate::message::stomp_message::Ownership;
  105|       |
  106|       |    use super::*;
  107|       |
  108|      1|    #[test]
  109|      1|    fn test_fixed_length_happy() {
  110|      1|        let mut p = FixedLengthBodyParser::new();
  111|      1|        let mut message = StompMessage::new(Ownership::Session);
  112|      1|
  113|      1|        p.reinit(23);
  114|      1|
  115|      1|        match p.push(b"abcdefghij", &mut message) {
  116|      1|            Ok(10) => {
  117|      1|                assert!(!p.is_done());
  118|       |            }
  119|      0|            Err(_) => panic!("parse error"),
  120|      0|            _ => panic!("unexpected"),
  121|       |        }
  122|      1|        match p.push(b"klmnopqrst", &mut message) {
  123|      1|            Ok(10) => {
  124|      1|                assert!(!p.is_done());
  125|       |            }
  126|      0|            Err(_) => panic!("parse error"),
  127|      0|            _ => panic!("unexpected"),
  128|       |        }
  129|      1|        match p.push(b"uvw\0", &mut message) {
  130|      1|            Ok(4) => {
  131|      1|                assert!(p.is_done());
  132|       |            }
  133|      0|            Err(_) => panic!("parse error"),
  134|      0|            _ => panic!("unexpected"),
  135|       |        }
  136|      1|    }
  137|       |
  138|      1|    #[test]
  139|      1|    fn test_fixed_length_unhappy() {
  140|      1|        let mut p = FixedLengthBodyParser::new();
  141|      1|        let mut message = StompMessage::new(Ownership::Session);
  142|      1|
  143|      1|        p.reinit(22); // <- bug
  144|      1|
  145|      1|        match p.push(b"abcdefghij", &mut message) {
  146|      1|            Ok(10) => {
  147|      1|                assert!(!p.is_done());
  148|       |            }
  149|      0|            Err(_) => panic!("parse error"),
  150|      0|            _ => panic!("unexpected"),
  151|       |        }
  152|      1|        match p.push(b"klmnopqrst", &mut message) {
  153|      1|            Ok(10) => {
  154|      1|                assert!(!p.is_done());
  155|       |            }
  156|      0|            Err(_) => panic!("parse error"),
  157|      0|            _ => panic!("unexpected"),
  158|       |        }
  159|      1|        match p.push(b"uvw\0", &mut message) {
  160|      1|            Err(ParserState::InvalidMessage) => {
  161|      1|                assert!(!p.is_done());
  162|      1|                match p.state {
  163|      1|                    State::Error => {
  164|      1|                        // all good
  165|      1|                    }
  166|       |                    _ => {
  167|      0|                        panic!("unexpected parser state")
  168|       |                    }
  169|       |                }
  170|       |            }
  171|       |            Ok(4) => {
  172|      0|                panic!("unexpected")
  173|       |            }
  174|      0|            _ => panic!("unexpected"),
  175|       |        }
  176|      1|    }
  177|       |}

src/body_parser/text.rs:
    1|       |use crate::body_parser::{BodyParser, ParserState};
    2|       |use crate::message::stomp_message::StompMessage;
    3|       |
    4|       |/// STOMP protocol push parser, parses message body.
    5|       |///
    6|       |/// readonly bytes pushed in from the network and copied into the message
    7|       |
    8|       |/// state of the loop
    9|      3|#[derive(Debug, PartialEq)]
   10|       |enum State {
   11|       |    Start,
   12|       |    /// expecting trailing \0
   13|       |    ReadingBody,
   14|       |    Done,
   15|       |}
   16|       |
   17|       |/// Body parser for binary data when we dont know the expected length of the data, data must be text so we can scan for \0
   18|       |pub struct TextBodyParser {
   19|       |    state: State,
   20|       |    read: usize,
   21|       |    max_len: usize,
   22|       |}
   23|       |
   24|       |/// reads up to \0
   25|       |impl TextBodyParser {
   26|      2|    pub fn new(max_len: usize) -> TextBodyParser {
   27|      2|        TextBodyParser {
   28|      2|            state: State::Start,
   29|      2|            read: 0,
   30|      2|            max_len,
   31|      2|        }
   32|      2|    }
   33|       |
   34|       |    /// how many bytes read form the stream this inlcudes \0
   35|      0|    pub fn bytes_read(&self) -> usize {
   36|      0|        self.read
   37|      0|    }
   38|       |
   39|       |    /// must be called before use
   40|      1|    pub fn reinit(&mut self, max_len: usize) {
   41|      1|        self.state = State::Start;
   42|      1|        self.read = 0;
   43|      1|        self.max_len = max_len;
   44|      1|    }
   45|       |}
   46|       |
   47|       |impl BodyParser for TextBodyParser {
   48|      4|    fn push(&mut self, buffer: &[u8], message: &mut StompMessage) -> Result<usize, ParserState> {
   49|      4|        let mut p: usize = 0;
   50|     35|        for ch in buffer {
   51|     35|            p += 1;
   52|     35|            if *ch == b'\0' {
   53|      1|                message.add_body(&buffer[0..p - 1]);
   54|      1|                self.state = State::Done;
   55|      1|                self.read += p;
   56|      1|                return Ok(p);
   57|     34|            }
   58|     34|            if self.read + p > self.max_len {
   59|      1|                self.read += p;
   60|      1|                return Err(ParserState::BodyFlup);
   61|     33|            }
   62|       |        }
   63|       |
   64|      2|        message.add_body(&buffer[0..p]);
   65|      2|        self.state = State::ReadingBody;
   66|      2|        self.read += p;
   67|      2|
   68|      2|        Ok(p)
   69|      4|    }
   70|       |
   71|      3|    fn is_done(&self) -> bool {
   72|      3|        State::Done == self.state
   73|      3|    }
   74|       |}
   75|       |
   76|       |#[cfg(test)]
   77|       |mod tests {
   78|       |    use crate::message::stomp_message::Ownership;
   79|       |
   80|       |    use super::*;
   81|       |
   82|      1|    #[test]
   83|      1|    fn test_text_happy() {
   84|      1|        let mut p = TextBodyParser::new(1048576);
   85|      1|        let mut m = StompMessage::new(Ownership::Session);
   86|      1|
   87|      1|        p.reinit(1048576);
   88|      1|
   89|      1|        match p.push(b"abcdefghij", &mut m) {
   90|      1|            Ok(10) => {
   91|      1|                assert!(!p.is_done());
   92|       |            }
   93|      0|            Err(_) => panic!("parse error"),
   94|      0|            _ => panic!("unexpected"),
   95|       |        }
   96|      1|        match p.push(b"klmnopqrst", &mut m) {
   97|      1|            Ok(10) => {
   98|      1|                assert!(!p.is_done());
   99|       |            }
  100|      0|            Err(_) => panic!("parse error"),
  101|      0|            _ => panic!("unexpected"),
  102|       |        }
  103|      1|        match p.push(b"uvw\0", &mut m) {
  104|      1|            Ok(4) => {
  105|      1|                assert!(p.is_done());
  106|       |            }
  107|      0|            Err(_) => panic!("parse error"),
  108|      0|            _ => panic!("unexpected"),
  109|       |        }
  110|       |
  111|      1|        println!("parsed text message {:?}", m.as_string());
  112|      1|    }
  113|       |
  114|      1|    #[test]
  115|      1|    fn test_text_unhappy() {
  116|      1|        let mut p = TextBodyParser::new(10);
  117|      1|        let mut m = StompMessage::new(Ownership::Session);
  118|      1|
  119|      1|        match p.push(b"abcdefghijk", &mut m) {
  120|      1|            Ok(_) => {
  121|      0|                panic!("should flup")
  122|       |            }
  123|      1|            Err(ParserState::BodyFlup) => {}
  124|      0|            _ => panic!("unexpected"),
  125|       |        }
  126|      1|    }
  127|       |}

src/config/config.rs:
    1|       |//! Loads `romp.toml` and provides access to properties to the running application.
    2|       |//! The crate contains a documented example configuration.  Where possible config is similar to `xtomp`.
    3|       |
    4|       |use log::*;
    5|       |
    6|       |use std::collections::HashMap;
    7|       |
    8|       |use crate::system::limits::increase_rlimit_nofile;
    9|       |use crate::util::some_string;
   10|       |use serde_derive::Deserialize;
   11|       |
   12|       |pub static CONFIG_FILE: &str = "romp.toml";
   13|       |
   14|     44|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RNvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtBa_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_9___VisitorNtB1H_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RINvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtBb_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB6_9___VisitorNtB1I_7Visitor9visit_seqpEBf_
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtB8_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1F_7Visitor9visit_u64pEBc_
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtB8_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1F_7Visitor11visit_bytespEBc_
  ------------------
  | Unexecuted instantiation: _RNvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtB7_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB2_14___FieldVisitorNtB1E_7Visitor9expecting
  ------------------
  | _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtB8_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1F_7Visitor9visit_strNtNtCs8EBejUHYFjW_4toml2de5ErrorEBc_:
  |   14|     22|#[derive(Deserialize, Debug)]
  ------------------
  | _RINvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtBb_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB6_9___VisitorNtB1I_7Visitor9visit_mapNtNtCs8EBejUHYFjW_4toml2de10MapVisitorEBf_:
  |   14|     23|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtBa_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1F_11deserializeINtNtB1H_5value23BorrowedStrDeserializerNtNtCs8EBejUHYFjW_4toml2de5ErrorEEBe_
  ------------------
  | _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtB5_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeQNtNtCs8EBejUHYFjW_4toml2de12DeserializerEB9_:
  |   14|      1|#[derive(Deserialize, Debug)]
  ------------------
  | _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config34__IMPL_DESERIALIZE_FOR_ServerConfigNtBa_12ServerConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1F_11deserializeNtNtCs8EBejUHYFjW_4toml2de15StrDeserializerEBe_:
  |   14|     22|#[derive(Deserialize, Debug)]
  ------------------
   15|       |#[serde(default)]
   16|       |pub struct ServerConfig {
   17|       |    pub name: String,
   18|       |    pub bind_address: String,
   19|       |    pub admin_bind_address: Option<String>,
   20|       |    pub ports: Vec<u16>,
   21|       |    pub admin_port: u16,
   22|       |    pub socket_addresses: Vec<String>,
   23|       |    pub login: Option<String>,
   24|       |    pub passcode: Option<String>,
   25|       |    pub user_file: Option<String>,
   26|       |    pub pid_file: String,
   27|       |    pub secret: Option<String>,
   28|       |    pub secret_timeout: i64,
   29|       |    /// enable administration filters, e.g. shutdown
   30|       |    pub enable_admin: bool,
   31|       |    /// enable GUI console filters
   32|       |    pub enable_console: bool,
   33|       |    pub max_message_size: usize,
   34|       |    pub max_headers: usize,
   35|       |    pub max_header_len: usize,
   36|       |    pub max_subs: usize,
   37|       |    pub rlimit_nofile: u64,
   38|       |    pub request_client_buffer: usize,
   39|       |    pub response_client_buffer: usize,
   40|       |    pub access_log: LevelFilter,
   41|       |
   42|       |    pub heart_beat_read: u32,
   43|       |    pub heart_beat_write_min: u32,
   44|       |    pub heart_beat_write_max: u32,
   45|       |
   46|       |    pub websockets: bool,
   47|       |    pub websockets_origin: Option<String>,
   48|       |    pub server_flags: Vec<String>,
   49|       |
   50|       |    pub destinations: HashMap<String, Destination>,
   51|       |    pub downstreams: HashMap<String, Downstream>,
   52|       |}
   53|       |
   54|       |impl Default for ServerConfig {
   55|      2|    fn default() -> Self {
   56|      2|        ServerConfig {
   57|      2|            name: "romp".to_string(),
   58|      2|            bind_address: "127.0.0.1".to_string(),
   59|      2|            admin_bind_address: None,
   60|      2|            ports: vec![61613],
   61|      2|            admin_port: 61616,
   62|      2|            socket_addresses: vec!["127.0.0.1:61613".to_string()],
   63|      2|            login: None,
   64|      2|            passcode: None,
   65|      2|            user_file: None,
   66|      2|            pid_file: "/run/romp/romp.pid".to_string(),
   67|      2|            secret: None,
   68|      2|            secret_timeout: 60000,
   69|      2|            enable_admin: false,
   70|      2|            enable_console: false,
   71|      2|            max_message_size: 1048576,
   72|      2|            max_headers: 10,
   73|      2|            max_header_len: 200,
   74|      2|            max_subs: 4,
   75|      2|            rlimit_nofile: 1048576,
   76|      2|            request_client_buffer: 4096,
   77|      2|            response_client_buffer: 200,
   78|      2|            access_log: LevelFilter::Info,
   79|      2|
   80|      2|            heart_beat_read: 120000,
   81|      2|            heart_beat_write_min: 60000,
   82|      2|            heart_beat_write_max: 180000,
   83|      2|
   84|      2|            websockets: true,
   85|      2|            websockets_origin: None,
   86|      2|            server_flags: vec![],
   87|      2|
   88|      2|            destinations: HashMap::default(),
   89|      2|            downstreams: HashMap::default(),
   90|      2|        }
   91|      2|    }
   92|       |}
   93|       |
   94|       |impl ServerConfig {
   95|      1|    pub fn init(&mut self) {
   96|      1|        self.socket_addresses.clear();
   97|      2|        for port in self.ports.iter() {
   98|      2|            let port_string = port.to_string();
   99|      2|
  100|      2|            let mut addr: String = self.bind_address.clone();
  101|      2|            // trusted port is only opened on localhost, or admin_bind_address
  102|      2|            if self.admin_port == *port {
  103|      0|                if let Some(admin_bind) = &self.admin_bind_address {
  104|      0|                    addr = admin_bind.clone();
  105|      0|                } else {
  106|      0|                    addr = String::from("127.0.0.1");
  107|      0|                }
  108|      2|            }
  109|       |
  110|      2|            addr.push_str(":");
  111|      2|            addr.push_str(port_string.as_str());
  112|      2|            self.socket_addresses.push(addr);
  113|       |        }
  114|       |
  115|      1|        if increase_rlimit_nofile(self.rlimit_nofile) == 0 {
  116|      0|            info!("nofile: {}", self.rlimit_nofile);
  117|       |        } else {
  118|      1|            warn!("nofile not set");
  119|       |        }
  120|       |
  121|     10|        for (_, dest) in self.destinations.iter_mut() {
  122|     10|            if dest.max_message_size == 0 {
  123|      0|                dest.max_message_size = self.max_message_size;
  124|     10|            }
  125|       |        }
  126|       |
  127|      1|        if let Some(secret) = &self.secret {
  128|      0|            if secret.len() < 12 {
  129|      0|                warn!("crap password: {}", secret);
  130|      0|            }
  131|      1|        }
  132|      1|    }
  133|       |
  134|      0|    pub fn name(&self) -> &String {
  135|      0|        &self.name
  136|      0|    }
  137|       |
  138|       |    /// Returns true if this server requires login
  139|      0|    pub fn requires_auth(&self) -> bool {
  140|      0|        if self.login.is_some() && self.passcode.is_some() {
  141|      0|            return true;
  142|      0|        }
  143|      0|        if self.user_file.is_some() {
  144|      0|            return true;
  145|      0|        }
  146|      0|        if self.secret.is_some() {
  147|      0|            return true;
  148|      0|        }
  149|      0|        return false;
  150|      0|    }
  151|       |
  152|      0|    pub fn requires_sha_auth(&self) -> bool {
  153|      0|        self.secret.is_some()
  154|      0|    }
  155|       |
  156|      0|    pub fn supports_simple_auth(&self) -> bool {
  157|      0|        self.login.is_some() && self.passcode.is_some()
  158|      0|    }
  159|       |
  160|      0|    pub fn supports_user_auth(&self) -> bool {
  161|      0|        self.user_file.is_some()
  162|      0|    }
  163|       |
  164|      0|    pub fn login(&self) -> &String {
  165|      0|        match &self.login {
  166|      0|            Some(login) => login,
  167|      0|            None => panic!("login not configured"),
  168|       |        }
  169|      0|    }
  170|       |
  171|      0|    pub fn passcode(&self) -> &String {
  172|      0|        match &self.passcode {
  173|      0|            Some(passcode) => passcode,
  174|      0|            None => panic!("passcode not configured"),
  175|       |        }
  176|      0|    }
  177|       |
  178|       |    /// Get configuration of a named destination
  179|      0|    pub fn destination(&self, name: &String) -> Option<&Destination> {
  180|      0|        self.destinations.get(name)
  181|      0|    }
  182|       |
  183|       |    /// Get configuration of a named donwstream destination
  184|      0|    pub fn downstream(&self, name: &String) -> Option<&Downstream> {
  185|      0|        self.downstreams.get(name)
  186|      0|    }
  187|       |}
  188|       |
  189|    107|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB8_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1D_7Visitor9visit_u64pEBc_
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB8_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1D_7Visitor11visit_bytespEBc_
  ------------------
  | Unexecuted instantiation: _RNvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB7_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB2_14___FieldVisitorNtB1C_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RNvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtBa_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_9___VisitorNtB1F_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB5_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de15StrDeserializerEB9_
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtBa_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1D_11deserializeNtNtCs8EBejUHYFjW_4toml2de25DatetimeFieldDeserializerEBe_
  ------------------
  | _RINvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtBb_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB6_9___VisitorNtB1G_7Visitor9visit_mapNtNtCs8EBejUHYFjW_4toml2de10MapVisitorEBf_:
  |  189|     59|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtBa_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1D_11deserializeINtNtB1F_5value23BorrowedStrDeserializerNtNtCs8EBejUHYFjW_4toml2de5ErrorEEBe_
  ------------------
  | _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB5_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de10MapVisitorEB9_:
  |  189|     10|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB5_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de17ValueDeserializerEB9_
  ------------------
  | _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtBa_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1D_11deserializeNtNtCs8EBejUHYFjW_4toml2de15StrDeserializerEBe_:
  |  189|     49|#[derive(Deserialize, Debug)]
  ------------------
  | _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config33__IMPL_DESERIALIZE_FOR_DestinationNtB8_11DestinationNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1D_7Visitor9visit_strNtNtCs8EBejUHYFjW_4toml2de5ErrorEBc_:
  |  189|     49|#[derive(Deserialize, Debug)]
  ------------------
  190|       |#[serde(default)]
  191|       |pub struct Destination {
  192|       |    pub name: String,
  193|       |    pub max_connections: usize,
  194|       |    pub max_messages: usize,
  195|       |    pub expiry: u64,
  196|       |    pub pedantic_expiry: bool,
  197|       |    pub filter: bool,
  198|       |    pub filter_self: bool,
  199|       |    pub sub_by_user: bool,
  200|       |    pub auto_ack: bool,
  201|       |    pub max_message_size: usize,
  202|       |    pub min_delivery: usize,
  203|       |    pub write_block: bool,
  204|       |    pub read_block: bool,
  205|       |    pub web_write_block: bool,
  206|       |    pub web_read_block: bool,
  207|       |    pub stats: bool,
  208|       |    // pub log_messages: bool,
  209|       |}
  210|       |
  211|       |impl Destination {}
  212|       |
  213|       |impl Default for Destination {
  214|     15|    fn default() -> Self {
  215|     15|        Destination {
  216|     15|            name: "".to_string(),
  217|     15|            max_connections: 1000000,
  218|     15|            max_messages: 100,
  219|     15|            expiry: 120000,
  220|     15|            pedantic_expiry: false,
  221|     15|            filter: true,
  222|     15|            filter_self: false,
  223|     15|            sub_by_user: false,
  224|     15|            // default when init() is called is the server's max_message_size
  225|     15|            auto_ack: false,
  226|     15|            max_message_size: 1048576,
  227|     15|            min_delivery: 0,
  228|     15|            write_block: false,
  229|     15|            read_block: false,
  230|     15|            web_write_block: false,
  231|     15|            web_read_block: false,
  232|     15|            stats: false,
  233|     15|        }
  234|     15|    }
  235|       |}
  236|       |
  237|      0|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB8_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1B_7Visitor11visit_bytespEBc_
  ------------------
  | Unexecuted instantiation: _RNvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB7_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB2_14___FieldVisitorNtB1A_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RNvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtBa_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_9___VisitorNtB1D_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB8_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1B_7Visitor9visit_u64pEBc_
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB5_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de17ValueDeserializerEB9_
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtBa_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1B_11deserializeNtNtCs8EBejUHYFjW_4toml2de25DatetimeFieldDeserializerEBe_
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB5_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de15StrDeserializerEB9_
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtBa_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1B_11deserializeNtNtCs8EBejUHYFjW_4toml2de15StrDeserializerEBe_
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtBa_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1B_11deserializeINtNtB1D_5value23BorrowedStrDeserializerNtNtCs8EBejUHYFjW_4toml2de5ErrorEEBe_
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config6config32__IMPL_DESERIALIZE_FOR_DownstreamNtB5_10DownstreamNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtNtCs8EBejUHYFjW_4toml2de10MapVisitorEB9_
  ------------------
  238|       |#[serde(default)]
  239|       |pub struct Downstream {
  240|       |    pub name: String,
  241|       |    pub address: String,
  242|       |    pub login: Option<String>,
  243|       |    pub passcode: String,
  244|       |    pub destinations: Vec<String>,
  245|       |    pub local_destinations: Vec<String>,
  246|       |}
  247|       |
  248|       |impl Downstream {
  249|       |    // TODO shuly this is not necessary
  250|       |
  251|      0|    pub fn copy(&self) -> Downstream {
  252|      0|        Downstream {
  253|      0|            name: self.name.clone(),
  254|      0|            address: self.address.clone(),
  255|      0|            login: self.login.clone(),
  256|      0|            passcode: self.passcode.clone(),
  257|      0|            destinations: self.destinations.clone(),
  258|      0|            local_destinations: self.local_destinations.clone(),
  259|      0|        }
  260|      0|    }
  261|       |}
  262|       |
  263|       |impl Default for Downstream {
  264|      0|    fn default() -> Self {
  265|      0|        Downstream {
  266|      0|            name: "".to_string(),
  267|      0|            address: "127.0.0.1:61614".to_string(),
  268|      0|            login: some_string("xtomp"),
  269|      0|            passcode: "password".to_string(),
  270|      0|            destinations: vec![],
  271|      0|            local_destinations: vec![],
  272|      0|        }
  273|      0|    }
  274|       |}
  275|       |#[cfg(test)]
  276|       |mod tests {
  277|       |
  278|       |    use crate::init::CONFIG;
  279|       |
  280|      1|    #[test]
  281|      1|    fn test_init() {
  282|      1|        // println does not work here ?? it does in other code
  283|      1|        println!("CONFIG.socket_addr {:?}", CONFIG.socket_addresses);
  284|      1|        println!("CONFIG.login {:?}", CONFIG.login);
  285|      1|        println!("CONFIG.login {:?}", CONFIG.destinations);
  286|      1|    }
  287|       |}

src/config/log.rs:
    1|       |use std::path::Path;
    2|       |
    3|       |use log::*;
    4|       |
    5|      1|pub fn init_log4rs() {
    6|      1|    // TODO when this is released https://github.com/estk/log4rs/pull/73 make -v output debug level
    7|      1|    let mut log4rs_path = String::from("./conf");
    8|      1|    if let Ok(var) = std::env::var("ROMP_CONF") {
    9|      0|        log4rs_path = String::from(var.as_str());
   10|      1|    }
   11|      1|    log4rs_path.push_str("/romp-log4rs.yaml");
   12|      1|
   13|      1|    if Path::new(log4rs_path.as_str()).exists() {
   14|      1|        log4rs::init_file(log4rs_path, Default::default()).unwrap();
   15|      1|        info!("loading...");
   16|      0|    } else if Path::new("/etc/romp/romp-log4rs.yaml").exists() {
   17|      0|        log4rs::init_file(Path::new("/etc/romp/romp-log4rs.yaml"), Default::default()).unwrap();
   18|      0|        info!("loading...");
   19|      0|    }
   20|      1|}

src/config/user.rs:
    1|       |//! Loads and provides access to the data in the `romp.passwd` file.
    2|       |
    3|       |use std::collections::HashMap;
    4|       |use std::fs::File;
    5|       |use std::io::{prelude::*, BufReader};
    6|       |
    7|       |use log::*;
    8|       |
    9|       |use serde_derive::Deserialize;
   10|       |use sha2::{Digest, Sha256};
   11|       |
   12|       |/// read a properties file of name:pass_hash
   13|       |
   14|      0|#[derive(Deserialize, Debug)]
  ------------------
  | Unexecuted instantiation: _RINvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtBb_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB6_9___VisitorNtB1C_7Visitor9visit_mappEBf_
  ------------------
  | Unexecuted instantiation: _RINvXs_NvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtBa_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_7___FieldB1z_11deserializepEBe_
  ------------------
  | Unexecuted instantiation: _RINvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtBb_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB6_9___VisitorNtB1C_7Visitor9visit_seqpEBf_
  ------------------
  | Unexecuted instantiation: _RNvXNvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtB7_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB2_14___FieldVisitorNtB1y_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtB8_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1z_7Visitor9visit_u64pEBc_
  ------------------
  | Unexecuted instantiation: _RNvXs0_NvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtBa_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB5_9___VisitorNtB1B_7Visitor9expecting
  ------------------
  | Unexecuted instantiation: _RINvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtB5_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializepEB9_
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtB8_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1z_7Visitor11visit_bytespEBc_
  ------------------
  | Unexecuted instantiation: _RINvXNvXNvNtNtCs9SJoeSWam1h_4romp6config4user32__IMPL_DESERIALIZE_FOR_UserConfigNtB8_10UserConfigNtNtCs8EH9SFu2raK_5serde2de11Deserialize11deserializeNtB3_14___FieldVisitorNtB1z_7Visitor9visit_strpEBc_
  ------------------
   15|       |#[serde(default)]
   16|       |pub struct UserConfig {
   17|       |    // map from username to sha256 of password + admin flag
   18|       |    users: HashMap<String, (Box<[u8]>, bool)>,
   19|       |}
   20|       |
   21|       |impl Default for UserConfig {
   22|      1|    fn default() -> Self {
   23|      1|        UserConfig {
   24|      1|            users: HashMap::default(),
   25|      1|        }
   26|      1|    }
   27|       |}
   28|       |
   29|       |impl UserConfig {
   30|      1|    pub fn new() -> UserConfig {
   31|      1|        UserConfig {
   32|      1|            ..Default::default()
   33|      1|        }
   34|      1|    }
   35|       |
   36|       |    /// return the number of configured users
   37|      0|    pub fn len(&self) -> usize {
   38|      0|        self.users.len()
   39|      0|    }
   40|       |
   41|      1|    pub fn init(&mut self, user_file: &String) {
   42|      1|        debug!("loading...");
   43|      1|        let file = File::open(user_file).expect(format!("file not found {}", user_file).as_str());
   44|      1|        let buf = BufReader::new(file);
   45|     28|        buf.lines().map(|line| line.unwrap()).for_each(|line| {
   46|     28|            // discard blank line and comments
   47|     28|            if line.len() == 0 || line.chars().next().unwrap() == '#' {
   48|     14|                return;
   49|     14|            }
   50|     14|
   51|     14|            // split name:hex
   52|     14|            let mut parts = line.split(':');
   53|       |
   54|     14|            if let (Some(name), Some(hex)) = (parts.next(), parts.next()) {
   55|     14|                if !sanename::validate_sanename_word(name) {
   56|      0|                    info!("failed to load line \"{}\"", line);
   57|      0|                    return;
   58|     14|                }
   59|     14|                let admin;
   60|     14|                match parts.next() {
   61|      1|                    Some(flags) => admin = flags.eq("true"),
   62|     13|                    _ => admin = false,
   63|       |                }
   64|     14|                self.users.insert(
   65|     14|                    String::from(name.trim()),
   66|     14|                    (self.decode_hex(hex.trim()), admin),
   67|     14|                );
   68|       |            } else {
   69|      0|                info!("failed to load line \"{}\"", line);
   70|       |            }
   71|     28|        });
   72|      1|
   73|      1|        info!("loaded {} users from {}", self.users.len(), user_file);
   74|      1|    }
   75|       |
   76|       |    /// validate the password, against the sha256 has stored in the user file
   77|       |    /// returns (password_ok, is_admin)
   78|       |    pub fn check_passcode(&self, login: &str, passcode: &str) -> (bool, bool) {
   79|      7|        if let Some(bin) = self.users.get(login) {
   80|       |            // sha256(passcode)
   81|      7|            let mut sha256 = Sha256::new();
   82|      7|            sha256.write(passcode.as_bytes()).unwrap();
   83|      7|            sha256.write(b"\n").unwrap();
   84|      7|            // compare iterators over u8
   85|      7|            return (sha256.finalize().iter().eq(bin.0.iter()), bin.1);
   86|      0|        }
   87|      0|        debug!("login not found {}", login);
   88|      0|        (false, false)
   89|      7|    }
   90|       |
   91|       |    /// decode typical sha256 hex string or panic
   92|     14|    fn decode_hex(&self, hex: &str) -> Box<[u8]> {
   93|     14|        let decoded = base16::decode(hex).unwrap_or_else(|_| panic!("not a hex value"));
   94|     14|        if decoded.len() != 256 / 8 {
   95|      0|            panic!("not a sha256 hash");
   96|     14|        }
   97|     14|        let mut hash: Box<[u8]> = Box::new([0u8; 256 / 8]);
   98|     14|        hash.copy_from_slice(decoded.as_slice());
   99|     14|        hash
  100|     14|    }
  101|       |}
  102|       |
  103|       |#[cfg(test)]
  104|       |mod tests {
  105|       |    use super::*;
  106|       |
  107|      1|    #[test]
  108|      1|    fn test_init() {
  109|      1|        let mut user_config = UserConfig::new();
  110|      1|        user_config.init(&String::from("conf/romp.passwd"));
  111|      1|
  112|      1|        // bob & alice, despite all their years working in infosec, still use common passwords in test code.
  113|      1|
  114|      1|        assert!(user_config.check_passcode("bob", "password").0);
  115|      1|        assert!(user_config.check_passcode("alice", "password").0);
  116|       |
  117|      1|        assert!(!user_config.check_passcode("bob", "dE4Rfd").0);
  118|      1|        assert!(!user_config.check_passcode("alice", "ESIKEDSDD").0);
  119|       |
  120|      1|        assert!(user_config.check_passcode("admin2", "password").0);
  121|      1|        assert!(user_config.check_passcode("admin2", "password").1);
  122|      1|        assert!(!user_config.check_passcode("bob", "password").1);
  123|      1|    }
  124|       |}

src/lib.rs:
    1|      1|//! # Romp//! # Romp
    2|       |//!
    3|       |//! `romp` is a messaging server that uses the STOMP protocol, and STOMP over WebSockets.
    4|       |//!
    5|       |//! `romp` can run as a standalone server, hosting queues and/or topics to which clients can subscribe or
    6|       |//! send messages.   It can also host bespoke filters (handlers) to programmatically respond to incoming messages.
    7|       |//! This enables romp servers to act as app servers, using asynchronous messaging model.
    8|       |//!
    9|       |//!
   10|       |//! Custom filters can be added as follow...
   11|       |//!
   12|       |//! ```
   13|       |//!
   14|       |//!     // Filter structs have an init() method
   15|       |//!     use romp::bootstrap::romp_bootstrap;
   16|       |//!     use romp::workflow::filter::ping::PingFilter;
   17|       |//!     use romp::prelude::{romp_stomp_filter, romp_http_filter, romp_stomp, romp_http};
   18|       |//!     use romp::workflow::filter::http::hello::HelloFilter;
   19|       |//!     use romp::message::response::{get_response_ok, get_http_ok_msg};
   20|       |//!     use std::sync::Arc;
   21|       |//!
   22|       |//!     romp_stomp_filter(Box::new(PingFilter {}));
   23|       |//!
   24|       |//!     // Http Filter structs different error handling
   25|       |//!     romp_http_filter(Box::new(HelloFilter {}));
   26|       |//!
   27|       |//!     // STOMP filters can be written as closures
   28|       |//!     romp_stomp(|ctx, msg| {
   29|       |//!         if let Some(hdr) = msg.get_header("special-header") {
   30|       |//!             let mut session = ctx.session.write().unwrap();
   31|       |//!             session.send_message(get_response_ok("howdy".to_string()));
   32|       |//!             return Ok(true);
   33|       |//!         }
   34|       |//!         Ok(false)
   35|       |//!     });
   36|       |//!
   37|       |//!     // as can HTTP filters
   38|       |//!     romp_http(|ctx, msg| {
   39|       |//!         if "/pingerooni".eq(ctx.attributes.get("romp.uri").unwrap()) {
   40|       |//!             let mut session = ctx.session.write().unwrap();
   41|       |//!             session.send_message(Arc::new(get_http_ok_msg()));
   42|       |//!             return Ok(true);
   43|       |//!         }
   44|       |//!         Ok(false)
   45|       |//!     });
   46|       |//!
   47|       |//!     // Then bootstrap the server, this is commented out in the docs, because of some bug in the cargo test when this comment is uncommented.
   48|       |//!     // There is no bug in the code below its used like that in main.rs that compiles.
   49|       |//!     // tokio::run(bootstrap_romp());
   50|       |//!
   51|       |//! ```
   52|       |
   53|       |extern crate base64;
   54|       |#[macro_use]
   55|       |extern crate lazy_static;
   56|       |extern crate futures;
   57|       |extern crate log;
   58|       |extern crate log4rs;
   59|       |extern crate sanename;
   60|       |extern crate sha2;
   61|       |
   62|       |pub mod body_parser;
   63|       |pub mod bootstrap;
   64|       |pub mod config;
   65|       |pub mod downstream;
   66|       |pub mod errors;
   67|       |pub mod init;
   68|       |pub mod message;
   69|       |pub mod parser;
   70|       |pub mod persist;
   71|       |pub mod prelude;
   72|       |pub mod session;
   73|       |pub mod system;
   74|       |pub mod util;
   75|       |pub mod web_socket;
   76|       |pub mod workflow;

src/main.rs:
    1|      1|use romp::prelude::*;use romp::prelude::*;
    2|       |
    3|       |/// Romp server entry point
    4|       |///
    5|       |/// Custom filters can be added as follow...
    6|       |///
    7|       |/// ```
    8|       |///
    9|       |///     // Filter structs have an init() method
   10|       |///     romp_stomp_filter(Box::new(PingFilter {}));
   11|       |///
   12|       |///     // Http Filter structs different error handling
   13|       |///     romp_http_filter(Box::new(HelloFilter {}));
   14|       |///
   15|       |///     // STOMP filters can be written as closures
   16|       |///     romp_stomp(|ctx, msg| {
   17|       |///         if let Some(hdr) = msg.get_header("special-header") {
   18|       |///             let mut session = ctx.session.write().unwrap();
   19|       |///             session.send_message(get_response_ok("howdy".to_string()));
   20|       |///             return Ok(true);
   21|       |///         }
   22|       |///         Ok(false)
   23|       |///     });
   24|       |///
   25|       |///     // as can HTTP filters
   26|       |///     romp_http(|ctx, msg| {
   27|       |///         if "/pingerooni".eq(ctx.attributes.get("romp.uri").unwrap()) {
   28|       |///             let mut session = ctx.session.write().unwrap();
   29|       |///             session.send_message(Arc::new(get_http_ok_msg()));
   30|       |///             return Ok(true);
   31|       |///         }
   32|       |///         Ok(false)
   33|       |///     });
   34|       |///     ///
   35|       |///
   36|       |/// ```
   37|       |///
   38|       |/// Then bootstrap the server
   39|       |///
   40|       |/// ```
   41|       |/// tokio::run(bootstrap_romp());
   42|       |/// ```
   43|       |
   44|      0|fn main() {
   45|      0|    tokio::run(romp_bootstrap());
   46|      0|}

src/message/response.rs:
    1|       |//! Contains canned responses.
    2|       |
    3|       |use std::sync::Arc;
    4|       |
    5|       |use crate::errors::*;
    6|       |use crate::message::stomp_message::MessageType::Http;
    7|       |use crate::message::stomp_message::StompCommand;
    8|       |use crate::message::stomp_message::*;
    9|       |
   10|       |pub const SERVER_TAG: &str = "romp/0.4";
   11|       |
   12|       |/// Creates reply messages
   13|       |///
   14|       |/// TODO could optimize some of this to static strings
   15|       |/// TODO should have C style function prefixes e.g. msg_xxx
   16|       |
   17|      0|pub fn get_response_connected(ecg_hdr: Header) -> Arc<StompMessage> {
   18|      0|    let mut message = StompMessage::new(Ownership::Session);
   19|      0|    message.command = StompCommand::Connected;
   20|      0|    message.add_header("server", "romp");
   21|      0|    message.add_header("version", "1.0");
   22|      0|    message.push_header(ecg_hdr);
   23|      0|    Arc::new(message)
   24|      0|}
   25|       |
   26|      0|pub fn get_response_error_general() -> Arc<StompMessage> {
   27|      0|    let mut message = StompMessage::new(Ownership::Session);
   28|      0|    message.command = StompCommand::Error;
   29|      0|    message.add_header("message", "general");
   30|      0|    Arc::new(message)
   31|      0|}
   32|       |
   33|      0|pub fn get_response_error_server(_err: ServerError) -> Arc<StompMessage> {
   34|      0|    let mut message = StompMessage::new(Ownership::Session);
   35|      0|    message.command = StompCommand::Error;
   36|      0|    message.add_header("message", "general");
   37|      0|    Arc::new(message)
   38|      0|}
   39|       |
   40|      0|pub fn get_response_error(text: &str) -> Arc<StompMessage> {
   41|      0|    let mut message = StompMessage::new(Ownership::Session);
   42|      0|    message.command = StompCommand::Error;
   43|      0|    message.add_header_clone("message", text);
   44|      0|    Arc::new(message)
   45|      0|}
   46|       |
   47|      0|pub fn get_response_receipt(id: &str) -> Arc<StompMessage> {
   48|      0|    let mut message = StompMessage::new(Ownership::Session);
   49|      0|    message.command = StompCommand::Receipt;
   50|      0|    message.add_header_clone("receipt-id", id);
   51|      0|    Arc::new(message)
   52|      0|}
   53|       |
   54|      0|pub fn get_response_error_client(err: ClientError) -> Arc<StompMessage> {
   55|      0|    let mut message = StompMessage::new(Ownership::Session);
   56|      0|    message.command = StompCommand::Error;
   57|      0|    match err {
   58|      0|        ClientError::Syntax => {
   59|      0|            message.add_header("message", "syntax");
   60|      0|        }
   61|      0|        ClientError::BodyFlup => {
   62|      0|            message.add_header("message", "msg flup");
   63|      0|        }
   64|      0|        ClientError::HdrFlup => {
   65|      0|            message.add_header("message", "hdr flup");
   66|      0|        }
   67|      0|        ClientError::Timeout => {
   68|      0|            message.add_header("message", "timeout");
   69|      0|        }
   70|      0|        ClientError::SubsFlup => {
   71|      0|            message.add_header("message", "subs flup");
   72|      0|        }
   73|       |    }
   74|      0|    Arc::new(message)
   75|      0|}
   76|       |
   77|       |// STONP message with message header and no body
   78|      0|pub fn get_response_ok(text: String) -> Arc<StompMessage> {
   79|      0|    let mut message = StompMessage::new(Ownership::Session);
   80|      0|    message.command = StompCommand::Message;
   81|      0|    message.add_header_clone("message", text.as_str());
   82|      0|    message.add_body("\n".as_bytes());
   83|      0|    Arc::new(message)
   84|      0|}
   85|       |
   86|       |/// Get destination statistics message
   87|      0|pub fn get_response_stats_msg(
   88|      0|    destination_name: &str,
   89|      0|    size: usize,
   90|      0|    q_size: usize,
   91|      0|    delta: usize,
   92|      0|    total: usize,
   93|      0|) -> StompMessage {
   94|      0|    let mut message = StompMessage::new(Ownership::Session);
   95|      0|    message.command = StompCommand::Message;
   96|      0|    message.add_header("destination", "/xtomp/stat");
   97|      0|    message.add_body(
   98|      0|        format!(
   99|      0|            "{{\n  \"dest\":\"{}\",\n  \"sz\":{},\n  \"q\":{},\n  \"Δ\":{},\n  \"Σ\":{}\n}}\n",
  100|      0|            destination_name, size, q_size, delta, total
  101|      0|        )
  102|      0|        .as_bytes(),
  103|      0|    );
  104|      0|    message
  105|      0|}
  106|       |
  107|       |/// Get concurrent_connections statistics message
  108|      0|pub fn get_response_cc_msg(
  109|      0|    concurrent_connections: usize,
  110|      0|    uptime: i64,
  111|      0|    total_connections: usize,
  112|      0|    total_messages: usize,
  113|      0|) -> StompMessage {
  114|      0|    let mut message = StompMessage::new(Ownership::Session);
  115|      0|    message.command = StompCommand::Message;
  116|      0|    message.add_header("destination", "/xtomp/stat");
  117|      0|    message.add_body(format!("{{\n  \"cc\":{{\n    \"sz\":{},\n    \"up\":{},\n    \"Σ\":{},\n    \"Σµ\":{},\n    \"m\":{}\n  }}\n}}\n", concurrent_connections, uptime, total_connections, total_messages, 0).as_bytes());
  118|      0|    message
  119|      0|}
  120|       |
  121|       |// HTTP
  122|       |
  123|       |/// Get empty OK message
  124|      0|pub fn get_http_ok_msg() -> StompMessage {
  125|      0|    let mut message = StompMessage::new(Ownership::Session);
  126|      0|    message.message_type = Http;
  127|      0|    message.command = StompCommand::Ok;
  128|      0|    message.add_header("Access-Control-Allow-Origin", "*");
  129|      0|    message.add_header("Access-Control-Expose-Headers", "server");
  130|      0|    message.add_header("server", SERVER_TAG);
  131|      0|    message.add_header("content-length", "0");
  132|      0|    message
  133|      0|}
  134|       |
  135|       |/// Get text OK message
  136|      0|pub fn get_http_ok_text_content(text: String, content_type: &'static str) -> Arc<StompMessage> {
  137|      0|    let mut message = StompMessage::new(Ownership::Session);
  138|      0|    message.message_type = Http;
  139|      0|    message.command = StompCommand::Ok;
  140|      0|    message.add_header("Access-Control-Allow-Origin", "*");
  141|      0|    message.add_header("Access-Control-Expose-Headers", "server");
  142|      0|    message.add_header("server", SERVER_TAG);
  143|      0|    let bytes = text.as_bytes();
  144|      0|    message.push_header(Header::from_string(
  145|      0|        "content-length",
  146|      0|        bytes.len().to_string(),
  147|      0|    ));
  148|      0|    message.add_header("content-type", content_type);
  149|      0|    message.add_body(bytes);
  150|      0|    Arc::new(message)
  151|      0|}
  152|       |
  153|      0|pub fn get_http_err_msg() -> StompMessage {
  154|      0|    let mut message = StompMessage::new(Ownership::Session);
  155|      0|    message.message_type = Http;
  156|      0|    message.command = StompCommand::ServerError;
  157|      0|    message.add_header("server", SERVER_TAG);
  158|      0|    message
  159|      0|}
  160|       |
  161|      0|pub fn get_http_404_msg() -> StompMessage {
  162|      0|    let mut message = StompMessage::new(Ownership::Session);
  163|      0|    message.message_type = Http;
  164|      0|    message.command = StompCommand::NotFound;
  165|      0|    message.add_header("server", SERVER_TAG);
  166|      0|    message
  167|      0|}
  168|       |
  169|      0|pub fn get_http_503_msg() -> StompMessage {
  170|      0|    let mut message = StompMessage::new(Ownership::Session);
  171|      0|    message.message_type = Http;
  172|      0|    message.command = StompCommand::ServiceUnavailable;
  173|      0|    message.add_header("server", SERVER_TAG);
  174|      0|    message
  175|      0|}
  176|       |
  177|      0|pub fn get_http_client_err_msg() -> StompMessage {
  178|      0|    let mut message = StompMessage::new(Ownership::Session);
  179|      0|    message.message_type = Http;
  180|      0|    message.command = StompCommand::ClientError;
  181|      0|    message.add_header("server", SERVER_TAG);
  182|      0|    message
  183|      0|}
  184|       |
  185|      0|pub fn get_http_upgrade_msg(websocket_accept: &String) -> StompMessage {
  186|      0|    let mut message = StompMessage::new(Ownership::Session);
  187|      0|    message.message_type = Http;
  188|      0|    message.command = StompCommand::Upgrade;
  189|      0|    message.add_header("Upgrade", "websocket");
  190|      0|    message.add_header("Connection", "Upgrade");
  191|      0|    message.add_header("Sec-WebSocket-Protocol", "stomp");
  192|      0|    message.add_header_clone("Sec-WebSocket-Accept", websocket_accept);
  193|      0|    message
  194|      0|}

src/message/serializer.rs:
    1|       |//! Contains code for serializing StompMessage instances to memroy, either for
    2|       |//! delivery over the network or to the file system.
    3|       |
    4|       |use log::*;
    5|       |
    6|       |use crate::message::stomp_message::StompCommand;
    7|       |use crate::message::stomp_message::{Header, MessageType, StompMessage};
    8|       |use crate::session::mq::SessionMessage;
    9|       |use crate::web_socket::ws_response::{ws_write_frame_hdr, FrameType};
   10|       |
   11|       |/// serialize a StompMessage to a stream
   12|       |///
   13|       |/// fills an out buffer from Memory data
   14|       |/// pushes it to a stream checking each time how much actually got writ
   15|       |/// HTTP messages are similar enough that this struct also supports GET responses
   16|       |
   17|       |enum State {
   18|       |    WritingHeaders,
   19|       |    WritingBody,
   20|       |    WritingTerminator,
   21|       |}
   22|       |
   23|       |pub struct MessageSerializer {
   24|       |    message: Option<SessionMessage>,
   25|       |    state: State,
   26|       |    msg_pos: usize,
   27|       |    web_socket: bool,
   28|       |}
   29|       |
   30|       |impl MessageSerializer {
   31|      6|    pub fn new() -> MessageSerializer {
   32|      6|        MessageSerializer {
   33|      6|            message: None,
   34|      6|            state: State::WritingHeaders,
   35|      6|            msg_pos: 0,
   36|      6|            web_socket: false,
   37|      6|        }
   38|      6|    }
   39|       |
   40|      6|    pub fn is_serializing(&self) -> bool {
   41|      6|        self.message.is_some()
   42|      6|    }
   43|       |
   44|       |    /// Set a message to serialize.
   45|       |    /// # Panics
   46|       |    ///   function panics if previous message was not written out yet.
   47|      6|    pub fn set_message(&mut self, message: SessionMessage) {
   48|      6|        if self.is_serializing() {
   49|      0|            panic!("BUG set_message() should not be called until the previous message is finished")
   50|      6|        }
   51|      6|        self.state = State::WritingHeaders;
   52|      6|        self.msg_pos = 0;
   53|      6|        self.message = Some(message);
   54|      6|    }
   55|       |
   56|      0|    pub fn ws_upgrade(&mut self) {
   57|      0|        self.web_socket = true;
   58|      0|    }
   59|       |
   60|       |    /// Writes out the next chunk of data to the buffer.
   61|       |    ///
   62|       |    /// # Panics
   63|       |    ///   function panics if `set_message()` was not called first
   64|       |    ///
   65|       |    /// handles writing `\0` terminator
   66|     12|    pub fn write_chunk(&mut self, out: &mut Box<[u8]>) -> Result<usize, ()> {
   67|     12|        let message = &self.message.as_ref();
   68|     12|        let message = message.unwrap();
   69|     12|
   70|     12|        match &self.state {
   71|     12|            State::WritingHeaders => {
   72|       |                // up to 10 bytes needed for the Websockets frame header
   73|      6|                let mut ws_len: usize = 0;
   74|      6|                if self.web_socket {
   75|      2|                    ws_len = 10;
   76|      4|                }
   77|       |
   78|      6|                let hdr_len = ws_len + message.message.header_len() + message.session_hdr_len() + 1;
   79|      6|                if out.len() < hdr_len {
   80|      0|                    warn!("outgoing buffer flup");
   81|      0|                    drop(message);
   82|      0|                    self.message = None;
   83|      0|                    return Err(());
   84|      6|                }
   85|      6|
   86|      6|                let mut pos = 0;
   87|      6|
   88|      6|                if self.web_socket && MessageType::Http != message.message.message_type {
   89|      2|                    let frame_len = message.message.header_len()
   90|      2|                        + message.session_hdr_len()
   91|      2|                        + 1
   92|      2|                        + message.message.body_len()
   93|      2|                        + 1;
   94|      2|                    pos = ws_write_frame_hdr(out, frame_len, FrameType::FinText);
   95|      4|                }
   96|      6|                pos = self.write_common_hdrs(out, pos, &message.message);
   97|      6|
   98|      6|                pos = self.write_session_hdrs(out, pos, &message.headers);
   99|      6|
  100|      6|                out[pos] = b'\n';
  101|      6|                pos += 1;
  102|      6|                if message.message.body_len() == 0 {
  103|      3|                    if MessageType::Http == message.message.message_type {
  104|      0|                        // skip trailing \0
  105|      3|                    } else {
  106|      3|                        out[pos] = b'\0';
  107|      3|                        pos += 1;
  108|      3|                    }
  109|       |                    // set message to None
  110|      3|                    self.message.take();
  111|      3|                    return Ok(pos);
  112|       |                } else {
  113|      3|                    self.state = State::WritingBody;
  114|      3|                    return Ok(pos);
  115|       |                }
  116|       |            }
  117|       |            State::WritingBody => {
  118|      5|                let bytes = message.message.body_as_bytes();
  119|      5|                let pos = self.msg_pos;
  120|      5|                let mut remaining = bytes.len() - pos;
  121|      5|
  122|      5|                if remaining > out.len() {
  123|       |                    // add as much as possible
  124|      2|                    let len = out.len();
  125|      2|                    out[0..len].copy_from_slice(&bytes[pos..pos + len]);
  126|      2|                    self.msg_pos += len;
  127|      2|                    return Ok(len);
  128|      3|                } else if remaining == out.len() {
  129|       |                    // msg fits bang on but may still need to add \0
  130|      1|                    out[0..remaining].copy_from_slice(&bytes[pos..pos + remaining]);
  131|      1|                    self.msg_pos += remaining;
  132|      1|                    if StompCommand::Get == message.message.command {
  133|       |                        // skip trailing \0
  134|       |                        // set message to None
  135|      0|                        self.message.take();
  136|      0|                        return Ok(remaining);
  137|       |                    } else {
  138|      1|                        self.state = State::WritingTerminator;
  139|      1|                        return Ok(remaining);
  140|       |                    }
  141|       |                } else {
  142|       |                    // remaining and terminator fit
  143|      2|                    out[0..remaining].copy_from_slice(&bytes[pos..pos + remaining]);
  144|      2|                    self.msg_pos += remaining;
  145|      2|                    if StompCommand::Get == message.message.command {
  146|      0|                        // skip trailing \0
  147|      2|                    } else {
  148|      2|                        out[remaining] = b'\0';
  149|      2|                        remaining += 1;
  150|      2|                    }
  151|       |                    // set message to None
  152|      2|                    self.message.take();
  153|      2|                    return Ok(remaining);
  154|       |                }
  155|       |            }
  156|       |            State::WritingTerminator => {
  157|      1|                out[0] = b'\0';
  158|      1|                // set message to None
  159|      1|                self.message.take();
  160|      1|                return Ok(1);
  161|       |            }
  162|       |        }
  163|     12|    }
  164|       |
  165|       |    /// Write command and headers that are common to each copy of this message.
  166|       |    /// Returns new value of pos
  167|      6|    fn write_common_hdrs(&self, out: &mut Box<[u8]>, pos: usize, message: &StompMessage) -> usize {
  168|      6|        let mut pos = pos;
  169|      6|        let cmd = message.command.as_string().as_bytes();
  170|      6|        out[pos..pos + cmd.len()].copy_from_slice(cmd);
  171|      6|        pos += cmd.len();
  172|      6|        out[pos] = b'\n';
  173|      6|        pos += 1;
  174|       |
  175|      6|        for hdr in message.headers() {
  176|      6|            pos = self.write_hdr(out, pos, hdr)
  177|       |        }
  178|       |
  179|      6|        pos
  180|      6|    }
  181|       |
  182|       |    /// Writes headers that are specific to this session.
  183|       |    /// Returns new value of pos, i.e. total bytes written to out.
  184|      6|    fn write_session_hdrs(&self, out: &mut Box<[u8]>, pos: usize, headers: &Vec<Header>) -> usize {
  185|      6|        let mut pos = pos;
  186|      6|        for hdr in headers.iter() {
  187|      1|            pos = self.write_hdr(out, pos, hdr)
  188|       |        }
  189|       |
  190|      6|        pos
  191|      6|    }
  192|       |
  193|       |    /// Writes a header.
  194|       |    /// Return new value of pos.
  195|      7|    fn write_hdr(&self, out: &mut Box<[u8]>, pos: usize, hdr: &Header) -> usize {
  196|      7|        let mut pos = pos;
  197|      7|        let name_len = hdr.name.len();
  198|      7|        out[pos..pos + name_len].copy_from_slice(&hdr.name.as_bytes());
  199|      7|        pos += name_len;
  200|      7|
  201|      7|        out[pos] = b':';
  202|      7|        pos += 1;
  203|      7|
  204|      7|        let value_len = hdr.value.len();
  205|      7|        out[pos..pos + value_len].copy_from_slice(&hdr.value.as_bytes());
  206|      7|        pos += value_len;
  207|      7|
  208|      7|        out[pos] = b'\n';
  209|      7|        pos += 1;
  210|      7|
  211|      7|        pos
  212|      7|    }
  213|       |}
  214|       |
  215|       |#[cfg(test)]
  216|       |mod tests {
  217|       |
  218|       |    use super::*;
  219|       |
  220|       |    use std::sync::Arc;
  221|       |
  222|      1|    #[test]
  223|      1|    fn test_happy() {
  224|      1|        let mut message = StompMessage::new_send(b"some data", 0);
  225|      1|        message.add_header("some", "header");
  226|      1|        let message = Arc::new(message);
  227|      1|
  228|      1|        let mut serializer = MessageSerializer::new();
  229|      1|        serializer.set_message(SessionMessage {
  230|      1|            message,
  231|      1|            headers: vec![],
  232|      1|        });
  233|      1|
  234|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  235|      1|
  236|      1|        match serializer.write_chunk(&mut out) {
  237|      1|            Ok(sz) => {
  238|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  239|      1|                assert_eq!(sz, 18);
  240|       |            }
  241|      0|            Err(_sz) => {
  242|      0|                panic!("should still have body to write")
  243|       |            }
  244|       |        }
  245|      1|        match serializer.write_chunk(&mut out) {
  246|      0|            Err(_sz) => {
  247|      0|                panic!("we should finish")
  248|       |            }
  249|      1|            Ok(sz) => {
  250|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  251|      1|                assert_eq!(sz, 10);
  252|      1|                assert_eq!('\0', out[9] as char);
  253|       |            }
  254|       |        }
  255|      1|    }
  256|       |
  257|      1|    #[test]
  258|      1|    fn test_happy_multi_chunk_body() {
  259|      1|        let mut message = StompMessage::new_send(b"01234567012345670123", 0);
  260|      1|        message.add_header("some", "header");
  261|      1|        let message = Arc::new(message);
  262|      1|
  263|      1|        let mut serializer = MessageSerializer::new();
  264|      1|        serializer.set_message(SessionMessage {
  265|      1|            message,
  266|      1|            headers: vec![],
  267|      1|        });
  268|      1|
  269|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  270|      1|
  271|      1|        match serializer.write_chunk(&mut out) {
  272|      1|            Ok(sz) => {
  273|      1|                println!(
  274|      1|                    "chunk write {} '{}'",
  275|      1|                    sz,
  276|      1|                    String::from_utf8_lossy(&out[0..sz])
  277|      1|                );
  278|      1|                assert_eq!(sz, 18);
  279|       |            }
  280|      0|            Err(_sz) => {
  281|      0|                panic!("should still have body to write")
  282|       |            }
  283|       |        }
  284|       |
  285|      1|        let mut out: Box<[u8]> = Box::new([0; 8]);
  286|      1|        match serializer.write_chunk(&mut out) {
  287|      1|            Ok(sz) => {
  288|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  289|      1|                assert_eq!(sz, 8);
  290|      1|                assert_eq!('7', out[7] as char);
  291|       |            }
  292|      0|            Err(_sz) => {
  293|      0|                panic!("we should not finish")
  294|       |            }
  295|       |        }
  296|      1|        match serializer.write_chunk(&mut out) {
  297|      1|            Ok(sz) => {
  298|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  299|      1|                assert_eq!(sz, 8);
  300|      1|                assert_eq!('7', out[7] as char);
  301|       |            }
  302|      0|            Err(_sz) => {
  303|      0|                panic!("we should not finish")
  304|       |            }
  305|       |        }
  306|      1|        match serializer.write_chunk(&mut out) {
  307|      0|            Err(_sz) => {
  308|      0|                panic!("we should finish")
  309|       |            }
  310|      1|            Ok(sz) => {
  311|      1|                println!("chunk write");
  312|      1|                assert_eq!(sz, 5);
  313|      1|                assert_eq!('3', out[3] as char);
  314|      1|                assert_eq!('\0', out[4] as char);
  315|       |            }
  316|       |        }
  317|      1|    }
  318|       |
  319|      1|    #[test]
  320|      1|    fn test_happy_trailing_zero_as_chunk() {
  321|      1|        let mut message = StompMessage::new_send(b"some data", 0);
  322|      1|        message.add_header("some", "header");
  323|      1|        let message = Arc::new(message);
  324|      1|
  325|      1|        let mut serializer = MessageSerializer::new();
  326|      1|        serializer.set_message(SessionMessage {
  327|      1|            message,
  328|      1|            headers: vec![],
  329|      1|        });
  330|      1|
  331|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  332|      1|
  333|      1|        match serializer.write_chunk(&mut out) {
  334|      1|            Ok(sz) => {
  335|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  336|      1|                assert_eq!(sz, 18);
  337|       |            }
  338|      0|            Err(_sz) => {
  339|      0|                panic!("should still have body to write")
  340|       |            }
  341|       |        }
  342|       |
  343|      1|        let mut out: Box<[u8]> = Box::new([0; 9]);
  344|      1|        match serializer.write_chunk(&mut out) {
  345|      1|            Ok(sz) => {
  346|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  347|      1|                assert_eq!(sz, 9);
  348|      1|                assert_eq!('a', out[8] as char);
  349|       |            }
  350|      0|            Err(_sz) => {
  351|      0|                panic!("we should not finish")
  352|       |            }
  353|       |        }
  354|      1|        match serializer.write_chunk(&mut out) {
  355|      0|            Err(_sz) => {
  356|      0|                panic!("we should finish")
  357|       |            }
  358|      1|            Ok(sz) => {
  359|      1|                println!("chunk write");
  360|      1|                assert_eq!(sz, 1);
  361|      1|                assert_eq!('\0', out[0] as char);
  362|       |            }
  363|       |        }
  364|      1|    }
  365|       |
  366|      1|    #[test]
  367|      1|    fn test_happy_no_body() {
  368|      1|        let mut message = StompMessage::new_send(b"", 0);
  369|      1|        message.add_header("some", "header");
  370|      1|        let message = Arc::new(message);
  371|      1|
  372|      1|        let mut serializer = MessageSerializer::new();
  373|      1|        serializer.set_message(SessionMessage {
  374|      1|            message,
  375|      1|            headers: vec![],
  376|      1|        });
  377|      1|
  378|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  379|      1|
  380|      1|        match serializer.write_chunk(&mut out) {
  381|      1|            Ok(sz) => {
  382|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  383|      1|                assert_eq!(sz, 19);
  384|       |                // assert body minus \0
  385|      1|                assert_eq!(
  386|      1|                    "SEND\nsome:header\n\n",
  387|      1|                    String::from_utf8_lossy(&out[0..sz - 1])
  388|      1|                );
  389|       |            }
  390|      0|            Err(_sz) => {
  391|      0|                panic!("returns Ok() even with no body")
  392|       |            }
  393|       |        }
  394|      1|    }
  395|       |
  396|      1|    #[test]
  397|      1|    fn test_happy_no_body_ws() {
  398|      1|        let mut message = StompMessage::new_send(b"", 0);
  399|      1|        message.add_header("some", "header");
  400|      1|        let message = Arc::new(message);
  401|      1|
  402|      1|        let mut serializer = MessageSerializer::new();
  403|      1|        serializer.set_message(SessionMessage {
  404|      1|            message,
  405|      1|            headers: vec![],
  406|      1|        });
  407|      1|        serializer.web_socket = true;
  408|      1|
  409|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  410|      1|
  411|      1|        match serializer.write_chunk(&mut out) {
  412|      1|            Ok(sz) => {
  413|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  414|      1|                assert_eq!(sz, 21);
  415|       |                // assert body minus leading ws protocol & trailing \0
  416|      1|                assert_eq!(
  417|      1|                    "SEND\nsome:header\n\n",
  418|      1|                    String::from_utf8_lossy(&out[2..sz - 1])
  419|      1|                );
  420|       |            }
  421|      0|            Err(_sz) => {
  422|      0|                panic!("returns Ok() even with no body")
  423|       |            }
  424|       |        }
  425|      1|    }
  426|       |
  427|      1|    #[test]
  428|      1|    fn test_happy_no_body_ws_extra_hdr() {
  429|      1|        let mut message = StompMessage::new_send(b"", 0);
  430|      1|        message.add_header("some", "header");
  431|      1|        let message = Arc::new(message);
  432|      1|
  433|      1|        let mut serializer = MessageSerializer::new();
  434|      1|        serializer.set_message(SessionMessage {
  435|      1|            message,
  436|      1|            headers: vec![Header::new("a", "b")],
  437|      1|        });
  438|      1|        serializer.web_socket = true;
  439|      1|
  440|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
  441|      1|
  442|      1|        match serializer.write_chunk(&mut out) {
  443|      1|            Ok(sz) => {
  444|      1|                println!("chunk write '{}'", String::from_utf8_lossy(&out[0..sz]));
  445|      1|                assert_eq!(sz, 25);
  446|       |                // assert body minus leading ws protocol & trailing \0
  447|      1|                assert_eq!(
  448|      1|                    "SEND\nsome:header\na:b\n\n",
  449|      1|                    String::from_utf8_lossy(&out[2..sz - 1])
  450|      1|                );
  451|       |            }
  452|      0|            Err(_sz) => {
  453|      0|                panic!("returns Ok() even with no body")
  454|       |            }
  455|       |        }
  456|      1|    }
  457|       |}

src/message/stomp_message.rs:
    1|       |//! Contains the StompMessage struct that is passed to all filters. This struct is also used internally
    2|       |//! to store messages in queues and topics, and in `mq` a connection specific queue or pending messages.
    3|       |
    4|       |use log::*;
    5|       |
    6|       |use std::slice::Iter;
    7|       |use std::sync::atomic::{AtomicUsize, Ordering};
    8|       |use std::time::Duration;
    9|       |
   10|       |use chrono::prelude::*;
   11|       |
   12|       |/// These ownership flags are copied from xtomp and are indicative of use but
   13|       |/// not validated or required.
   14|      6|#[derive(Debug, PartialEq)]
   15|       |pub enum Ownership {
   16|       |    /// Owned by a client, e.g. while reading the message
   17|       |    Parser,
   18|       |    /// Owned by a client, while processing the message
   19|       |    Session,
   20|       |    /// Owned by a destination (queue/topic)
   21|       |    Destination,
   22|       |    /// Owned by the server
   23|       |    Constant,
   24|       |}
   25|       |
   26|      8|#[derive(Debug, Clone, PartialEq)]
   27|       |pub enum MessageType {
   28|       |    Stomp,
   29|       |    Http,
   30|       |}
   31|       |
   32|     28|#[derive(Debug, Clone, PartialEq)]
   33|       |pub enum StompCommand {
   34|       |    Init,
   35|       |    /// not read yet
   36|       |    Unknown,
   37|       |    Ack,
   38|       |    Send,
   39|       |    Nack,
   40|       |    Begin,
   41|       |    Abort,
   42|       |    Error,
   43|       |    Stomp,
   44|       |    Commit,
   45|       |    Connect,
   46|       |    Message,
   47|       |    Receipt,
   48|       |    Subscribe,
   49|       |    Connected,
   50|       |    Disconnect,
   51|       |    Unsubscribe,
   52|       |    /// HTTP GET (used for upgrading an http connection to websockets)
   53|       |    Get,
   54|       |    Ok,
   55|       |    ServerError,
   56|       |    ServiceUnavailable,
   57|       |    ClientError,
   58|       |    NotFound,
   59|       |    Upgrade,
   60|       |}
   61|       |
   62|       |impl StompCommand {
   63|     25|    pub fn as_string(&self) -> &str {
   64|     25|        match &self {
   65|     25|            StompCommand::Init => "",
   66|      2|            StompCommand::Unknown => "",
   67|      4|            StompCommand::Ack => "ACK",
   68|     16|            StompCommand::Send => "SEND",
   69|      0|            StompCommand::Nack => "NACK",
   70|      0|            StompCommand::Begin => "BEGIN",
   71|      0|            StompCommand::Abort => "ABORT",
   72|      0|            StompCommand::Error => "ERROR",
   73|      0|            StompCommand::Stomp => "STOMP",
   74|      0|            StompCommand::Commit => "COMMIT",
   75|      0|            StompCommand::Connect => "CONNECT",
   76|      3|            StompCommand::Message => "MESSAGE",
   77|      0|            StompCommand::Receipt => "RECEIPT",
   78|      0|            StompCommand::Subscribe => "SUBSCRIBE",
   79|      0|            StompCommand::Connected => "CONNECTED",
   80|      0|            StompCommand::Disconnect => "DISCONNECT",
   81|      0|            StompCommand::Unsubscribe => "UNSUBSCRIBE",
   82|      0|            StompCommand::Get => "GET",
   83|      0|            StompCommand::Ok => "HTTP/1.1 200 OK",
   84|      0|            StompCommand::ServerError => "HTTP/1.1 500 Server Error",
   85|      0|            StompCommand::ServiceUnavailable => "HTTP/1.1 503 Service Unavailable",
   86|      0|            StompCommand::ClientError => "HTTP/1.1 400 Bad Request",
   87|      0|            StompCommand::NotFound => "HTTP/1.1 404 Not Found",
   88|      0|            StompCommand::Upgrade => "HTTP/1.1 101 Switching Protocols",
   89|       |        }
   90|     25|    }
   91|       |
   92|     11|    pub fn len(&self) -> usize {
   93|     11|        self.as_string().len()
   94|     11|    }
   95|       |}
   96|       |
   97|       |/// StompMessage is a command, a list of headers, and a body (typically text can be binary)
   98|       |/// Can be an incoming message or outgoing message.
   99|       |/// Typically StompMessage represents a STOMP protocol message but is also used to represent HTTP requests and responses.
  100|       |///
  101|       |/// Most STOMP commands have no body to the message, just command and headers.
  102|       |/// Headers in this struct contain all the headers received, or, for an outgoing message,
  103|       |/// all the common headers. Additional client specific headers may be added when serializing the message.
  104|       |///
  105|      6|#[derive(Debug)]
  106|       |pub struct StompMessage {
  107|       |    pub command: StompCommand,
  108|       |    pub message_type: MessageType,
  109|       |    pub id: usize,
  110|       |    pub owner: Ownership,
  111|       |    /// who to push message to None == all users
  112|       |    pub to: Option<String>,
  113|       |    pub timestamp: DateTime<Utc>,
  114|       |    // TODO u64 as millis might be more convenient
  115|       |    pub expiry: Duration,
  116|       |    // source of the message
  117|       |    pub session_id: Option<usize>,
  118|       |    delivered: AtomicUsize,
  119|       |    headers: Box<Vec<Header>>,
  120|       |    /// length of the header block, must be smaller than input buffer currently, excludes extra \n or message or trailing \0
  121|       |    hdr_len: usize,
  122|       |    /// list of [u8] chunks of body data, not \0 terminated
  123|       |    body: Box<Vec<Box<Vec<u8>>>>,
  124|       |    body_len: usize,
  125|       |}
  126|       |
  127|       |impl Default for StompMessage {
  128|     58|    fn default() -> Self {
  129|     58|        StompMessage {
  130|     58|            command: StompCommand::Unknown,
  131|     58|            owner: Ownership::Session,
  132|     58|            id: 0,
  133|     58|            to: None,
  134|     58|            timestamp: Utc::now(),
  135|     58|            expiry: Duration::new(60, 0),
  136|     58|            session_id: None,
  137|     58|            delivered: AtomicUsize::new(0),
  138|     58|            message_type: MessageType::Stomp,
  139|     58|            headers: Box::new(Vec::with_capacity(10)),
  140|     58|            hdr_len: 0,
  141|     58|            body: Box::new(Vec::new()),
  142|     58|            body_len: 0,
  143|     58|        }
  144|     58|    }
  145|       |}
  146|       |
  147|       |impl StompMessage {
  148|     41|    pub fn new(owner: Ownership) -> StompMessage {
  149|     41|        StompMessage {
  150|     41|            owner,
  151|     41|            ..Default::default()
  152|     41|        }
  153|     41|    }
  154|       |
  155|     15|    pub fn new_send(data: &[u8], id: usize) -> StompMessage {
  156|     15|        let mut message = StompMessage {
  157|     15|            command: StompCommand::Send,
  158|     15|            owner: Ownership::Session,
  159|     15|            id,
  160|     15|            ..Default::default()
  161|     15|        };
  162|     15|
  163|     15|        message.add_body(data);
  164|     15|        message
  165|     15|    }
  166|       |
  167|       |    /// take ownership of the message contents, a move, not a copy.
  168|       |    /// this instance will be reset to its init state
  169|       |    /// take() does not change session_id.
  170|      1|    pub fn take(&mut self, owner: Ownership) -> StompMessage {
  171|      1|        let mut message = StompMessage::new(owner);
  172|      1|        message.command = std::mem::replace(&mut self.command, StompCommand::Unknown);
  173|      1|        message.id = self.id;
  174|      1|        self.id = 0;
  175|      1|        if self.to.is_some() {
  176|      0|            message.to.replace(self.to.take().unwrap());
  177|      1|        } // else both stay None
  178|      1|        message.timestamp = self.timestamp;
  179|      1|        self.timestamp = Utc::now();
  180|      1|        message.expiry = self.expiry;
  181|      1|        self.expiry = Duration::new(60, 0);
  182|      1|        message.session_id = self.session_id;
  183|      1|        message.delivered = AtomicUsize::new(self.delivered.load(Ordering::Relaxed));
  184|      1|        self.delivered.store(0, Ordering::SeqCst);
  185|      1|        message.message_type = std::mem::replace(&mut self.message_type, MessageType::Stomp);
  186|      1|        message.headers = std::mem::replace(&mut self.headers, Box::new(Vec::with_capacity(10)));
  187|      1|        message.hdr_len = self.hdr_len;
  188|      1|        self.hdr_len = 0;
  189|      1|        message.body = std::mem::replace(&mut self.body, Box::new(Vec::new()));
  190|      1|        message.body_len = self.body_len;
  191|      1|        self.body_len = 0;
  192|      1|
  193|      1|        message
  194|      1|    }
  195|       |
  196|       |    /// clone the message contents, instance left unchanged
  197|      8|    pub fn clone(&self, owner: Ownership, id: usize) -> StompMessage {
  198|      8|        let mut message = StompMessage::new(owner);
  199|      8|        message.command = self.command.clone();
  200|      8|        message.id = id;
  201|      8|        message.to = self.to.clone();
  202|      8|        message.timestamp = self.timestamp.clone();
  203|      8|        message.expiry = self.expiry.clone();
  204|      8|        message.session_id = self.session_id.clone();
  205|      8|        message.delivered = AtomicUsize::new(self.delivered.load(Ordering::Relaxed));
  206|      8|        message.message_type = self.message_type.clone();
  207|      8|        message.headers = Box::new(Vec::with_capacity(self.headers.len()));
  208|      8|        for hdr in self.headers.iter() {
  209|      3|            message.headers.push(hdr.clone());
  210|      3|        }
  211|      7|        message.hdr_len = self.hdr_len;
  212|      7|        message.body = Box::new(Vec::with_capacity(1));
  213|      7|        message.body.push(self.combine());
  214|      7|        message.body_len = self.body_len;
  215|      7|
  216|      7|        message
  217|      7|    }
  218|       |
  219|       |    /// clone message setting the command to Message, (as it should be sent out)
  220|      8|    pub fn clone_to_message(&self, owner: Ownership, id: usize) -> StompMessage {
  221|      8|        let mut message = self.clone(owner, id);
  222|      8|        message.command = StompCommand::Message;
  223|      8|        message.headers.retain(|hdr| match hdr.name.as_str() {
  224|      3|            "receipt" | "ack" => false,
  225|      1|            _ => true,
  226|      8|        });
  227|      8|        message.add_header_clone("message-id", &id.to_string());
  228|      8|        message.recalculate_hdr_len();
  229|      8|
  230|      8|        message
  231|      8|    }
  232|       |
  233|       |    // create here so that it shares a lifetime with StompMessage
  234|     17|    pub fn add_header_clone(&mut self, name: &str, value: &str) {
  235|     17|        let hdr = Header {
  236|     17|            name: sanename::sanitize(name),
  237|     17|            // in the STOMP spec whitespace trimming is frowned upon
  238|     17|            value: String::from(value.trim()),
  239|     17|        };
  240|     17|        self.hdr_len += hdr.len() + 1; // the trailing \n
  241|     17|        self.headers.push(hdr);
  242|     17|    }
  243|       |
  244|       |    /// add header from rust code
  245|     56|    pub fn add_header(&mut self, name: &'static str, value: &'static str) {
  246|     56|        let hdr = Header {
  247|     56|            name: String::from(name),
  248|     56|            value: String::from(value),
  249|     56|        };
  250|     56|        self.hdr_len += hdr.len() + 1; // the trailing \n
  251|     56|        self.headers.push(hdr);
  252|     56|    }
  253|       |
  254|       |    /// same as add_header(str, str)
  255|      0|    pub fn push_header(&mut self, hdr: Header) {
  256|      0|        self.hdr_len += hdr.len() + 1; // the trailing \n
  257|      0|        self.headers.push(hdr);
  258|      0|    }
  259|       |
  260|       |    /// return header value
  261|     14|    pub fn get_header(&self, name: &str) -> Option<&String> {
  262|     17|        for hdr in self.headers.iter() {
  263|     17|            if name.eq(&hdr.name) {
  264|     14|                return Some(&hdr.value);
  265|      3|            }
  266|       |        }
  267|      0|        None
  268|     14|    }
  269|       |
  270|     24|    pub fn get_header_case_insensitive(&self, name: &str) -> Option<&String> {
  271|    121|        for hdr in self.headers.iter() {
  272|    121|            if name.eq_ignore_ascii_case(&hdr.name) {
  273|     22|                return Some(&hdr.value);
  274|     99|            }
  275|       |        }
  276|      2|        None
  277|     24|    }
  278|       |
  279|      0|    pub fn remove_header(&mut self, name: &str) {
  280|      0|        self.headers.retain(|hdr| !hdr.name.eq(name));
  281|      0|    }
  282|       |
  283|       |    pub fn extract_header(&mut self, name: &str) -> Option<Header> {
  284|      3|        if let Some(pos) = self.headers.iter().position(|hdr| hdr.name.eq(name)) {
  285|      1|            let hdr = self.headers.remove(pos);
  286|      1|            return Some(hdr);
  287|      1|        }
  288|      1|        None
  289|      2|    }
  290|       |
  291|      3|    pub fn count_headers(&self) -> usize {
  292|      3|        self.headers.len()
  293|      3|    }
  294|       |
  295|      0|    pub fn delivered(&self) -> usize {
  296|      0|        self.delivered.load(Ordering::SeqCst)
  297|      0|    }
  298|       |
  299|       |    /// increment delivered count and return previous value
  300|      9|    pub fn increment_delivered(&self, count: usize) -> usize {
  301|      9|        self.delivered.fetch_add(count, Ordering::SeqCst)
  302|      9|    }
  303|       |
  304|       |    /// return space required to render the headers block (excluding trailing \n or trailing \0)
  305|     11|    pub fn header_len(&self) -> usize {
  306|     11|        self.command.len() + 1 + self.hdr_len
  307|     11|    }
  308|       |
  309|      7|    fn recalculate_hdr_len(&mut self) {
  310|      7|        let mut len: usize = 0;
  311|      9|        for hdr in self.headers.iter() {
  312|      9|            len += hdr.name.len();
  313|      9|            len += 1;
  314|      9|            len += hdr.value.len();
  315|      9|            len += 1;
  316|      9|        }
  317|      8|        self.hdr_len = len;
  318|      8|    }
  319|       |
  320|      6|    pub fn headers(&self) -> Iter<Header> {
  321|      6|        self.headers.iter()
  322|      6|    }
  323|       |
  324|       |    /// return space required to render the body (excluding trailing \0)
  325|     24|    pub fn body_len(&self) -> usize {
  326|     24|        self.body_len
  327|     24|    }
  328|       |
  329|       |    /// return space required to render the whole message
  330|      0|    pub fn len(&self) -> usize {
  331|      0|        self.header_len() + 1 + self.body_len() + 1
  332|      0|    }
  333|       |
  334|     33|    pub fn add_body(&mut self, chunk: &[u8]) {
  335|     33|        // clone chunk
  336|     33|        self.body.push(Box::new(Vec::from(chunk)));
  337|     33|        self.body_len += chunk.len();
  338|     33|    }
  339|       |
  340|      9|    pub fn combine(&self) -> Box<Vec<u8>> {
  341|      9|        let mut body = Box::new(Vec::with_capacity(self.body_len));
  342|     14|        for chunk in self.body.iter() {
  343|     14|            body.extend_from_slice(chunk);
  344|     14|        }
  345|      9|        return body;
  346|      9|    }
  347|       |
  348|      1|    pub fn combine_chunks(&mut self) {
  349|      1|        let body = self.combine();
  350|      1|        self.body = Box::new(Vec::with_capacity(1));
  351|      1|        self.body_len = body.len();
  352|      1|        self.body.push(body);
  353|      1|    }
  354|       |
  355|       |    /// get headers as oned
  356|       |    /// String, not very efficient
  357|      8|    pub fn hdrs_as_string(&self) -> String {
  358|      8|        let mut hdrs: String = "".to_owned();
  359|      8|        hdrs.push_str(self.command.as_string());
  360|      8|        hdrs.push_str("\n");
  361|     11|        for hdr in self.headers.iter() {
  362|     11|            hdrs.push_str(hdr.name.as_str());
  363|     11|            hdrs.push_str(":");
  364|     11|            hdrs.push_str(hdr.value.as_str());
  365|     11|            hdrs.push_str("\n");
  366|     11|        }
  367|       |
  368|      8|        hdrs
  369|      8|    }
  370|       |
  371|       |    /// Lazy way to process the body. Not recommended since it clones the whole message
  372|       |    /// to an owned String via from_utf8_lossy()
  373|      6|    pub fn body_as_string(&self) -> String {
  374|      6|        let mut body: String = "".to_owned();
  375|     11|        for chunk in self.body.iter() {
  376|     11|            body.push_str(&String::from_utf8_lossy(chunk));
  377|     11|        }
  378|       |
  379|      6|        return body;
  380|      6|    }
  381|       |
  382|       |    /// Return a readonly pointer to the message contents as bytes.
  383|       |    /// This is the proper way to access a message received over the wire.
  384|       |    ///
  385|       |    /// # Panics
  386|       |    ///
  387|       |    /// If this is not a single chunk message, i.e. `combine()` was used.
  388|      5|    pub fn body_as_bytes(&self) -> &Box<Vec<u8>> {
  389|      5|        if self.body_len == 0 {
  390|      0|            error!("fatal: attempt to serialize zero len message");
  391|      0|            panic!("attempt to serialize zero len message");
  392|      5|        }
  393|      5|        if self.body.len() == 1 {
  394|      5|            let chunk = self.body.iter().next().unwrap();
  395|      5|            return chunk;
  396|      0|        }
  397|      0|        error!("fatal: attempt to serialize multi chunk message");
  398|      0|        panic!("attempt to serialize multi chunk message");
  399|      5|    }
  400|       |
  401|       |    /// for debug, real Send message need additional headers
  402|      1|    pub fn as_string(&self) -> String {
  403|      1|        let mut msg: String = "".to_owned();
  404|      1|        msg.push_str(self.hdrs_as_string().as_str());
  405|      1|        msg.push_str("\n");
  406|      1|        msg.push_str(self.body_as_string().as_str());
  407|      1|        match self.message_type {
  408|      1|            MessageType::Stomp => {
  409|      1|                msg.push_str("\0");
  410|      1|            }
  411|      0|            _ => {}
  412|       |        }
  413|       |
  414|      1|        return msg;
  415|      1|    }
  416|       |}
  417|       |
  418|     10|#[derive(Debug, Clone)]
  419|       |pub struct Header {
  420|       |    pub name: String,
  421|       |    pub value: String,
  422|       |}
  423|       |
  424|       |impl Header {
  425|      8|    pub fn new(name: &str, value: &str) -> Header {
  426|      8|        Header {
  427|      8|            name: String::from(name),
  428|      8|            value: String::from(value),
  429|      8|        }
  430|      8|    }
  431|      0|    pub fn from_string(name: &str, value: String) -> Header {
  432|      0|        Header {
  433|      0|            name: String::from(name),
  434|      0|            value,
  435|      0|        }
  436|      0|    }
  437|       |
  438|       |    /// return length of the header data presuming name:value representation with no spaces.
  439|     75|    pub fn len(&self) -> usize {
  440|     75|        self.name.len() + 1 + self.value.len()
  441|     75|    }
  442|       |}
  443|       |
  444|       |#[cfg(test)]
  445|       |mod tests {
  446|       |    use super::*;
  447|       |    use crate::message::stomp_message::MessageType::Http;
  448|       |
  449|      1|    #[test]
  450|      1|    fn test_init() {
  451|      1|        let mut message = StompMessage::new(Ownership::Session);
  452|      1|        message.command = StompCommand::Ack;
  453|      1|        message.add_header("name1", "value1");
  454|      1|        message.add_header("name2", "value2");
  455|      1|        message.add_body(b"{\"resp1\": true}\n");
  456|      1|
  457|      1|        println!("StompMessage {:?}", message);
  458|      1|
  459|      1|        println!(
  460|      1|            "serialize:: \n{}\n{}\0",
  461|      1|            message.hdrs_as_string(),
  462|      1|            message.body_as_string()
  463|      1|        );
  464|      1|
  465|      1|        assert_eq!(message.body_len, message.body.iter().next().unwrap().len());
  466|      1|    }
  467|       |
  468|      1|    #[test]
  469|      1|    fn test_chunks() {
  470|      1|        let mut message = StompMessage::new(Ownership::Session);
  471|      1|        message.command = StompCommand::Ack;
  472|      1|        message.add_header("name1", "value1");
  473|      1|        message.add_header("name2", "value2");
  474|      1|        message.add_body(b"{\n");
  475|      1|        message.add_body(b"  {\"resp1\": true},\n");
  476|      1|        message.add_body(b"  {\"resp1\": true},\n");
  477|      1|        message.add_body(b"  {\"resp1\": true}\n");
  478|      1|        message.add_body(b"}\n");
  479|      1|
  480|      1|        println!("StompMessage {:?}", message);
  481|      1|
  482|      1|        let mut len: usize = 0;
  483|      5|        for vec in message.body.iter() {
  484|      5|            len += vec.len();
  485|      5|        }
  486|      1|        assert_eq!(message.body_len(), len);
  487|       |
  488|      1|        println!(
  489|      1|            "serialize:: \n{}\n{}\0",
  490|      1|            message.hdrs_as_string(),
  491|      1|            message.body_as_string()
  492|      1|        );
  493|      1|
  494|      1|        message.combine_chunks();
  495|      1|
  496|      1|        assert_eq!(
  497|      1|            message.body_len(),
  498|      1|            message.body.iter().next().unwrap().len()
  499|      1|        );
  500|      1|        println!("StompMessage {:?}", message);
  501|      1|
  502|      1|        println!(
  503|      1|            "serialize:: \n{}\n{}\0",
  504|      1|            message.hdrs_as_string(),
  505|      1|            message.body_as_string()
  506|      1|        );
  507|      1|    }
  508|       |
  509|      1|    #[test]
  510|      1|    fn test_take() {
  511|      1|        let mut message = StompMessage::new(Ownership::Session);
  512|      1|        message.command = StompCommand::Ack;
  513|      1|        message.add_header("name1", "value1");
  514|      1|        message.add_header("name2", "value2");
  515|      1|        message.add_body(b"{\"resp1\": true}\n");
  516|      1|        assert_eq!(message.body_len(), 16);
  517|       |
  518|      1|        let copy = message.take(Ownership::Session);
  519|      1|
  520|      1|        assert_eq!(message.body_len(), 0);
  521|      1|        assert_eq!(copy.body_len(), 16);
  522|       |
  523|      1|        println!("Take Test orig='{:?}'", message);
  524|      1|        println!("Take Test copy='{:?}'", copy);
  525|      1|
  526|      1|        println!(
  527|      1|            "Take Test serialize empty:: \n{}\n{}\0",
  528|      1|            message.hdrs_as_string(),
  529|      1|            message.body_as_string()
  530|      1|        );
  531|      1|        println!(
  532|      1|            "Take Test serialize:: \n{}\n{}\0",
  533|      1|            copy.hdrs_as_string(),
  534|      1|            copy.body_as_string()
  535|      1|        );
  536|      1|    }
  537|       |
  538|      1|    #[test]
  539|      1|    fn test_clone_to_message() {
  540|      1|        let mut message = StompMessage::new(Ownership::Session);
  541|      1|        message.command = StompCommand::Send;
  542|      1|        message.add_header("receipt", "booya");
  543|      1|        message.add_header("ack", "client");
  544|      1|        message.add_header("name1", "value1");
  545|      1|        message.add_body(b"{\"resp1\": true}\n");
  546|      1|        assert_eq!(message.body_len(), 16);
  547|       |
  548|      1|        let copy = message.clone_to_message(Ownership::Destination, 23);
  549|      1|
  550|      1|        //        println!("Take Test orig='{:?}'", message);
  551|      1|        //        println!("Take Test copy='{:?}'", copy);
  552|      1|
  553|      1|        assert_eq!(message.body_len(), 16);
  554|      1|        assert_eq!(copy.body_len(), 16);
  555|       |        // customer hdr + message-id added
  556|      1|        assert_eq!(copy.count_headers(), 2);
  557|      1|        assert_eq!(copy.header_len(), 35);
  558|      1|        assert_eq!(copy.header_len(), copy.hdrs_as_string().len());
  559|      1|    }
  560|       |
  561|      1|    #[test]
  562|      1|    pub fn test_partial_eq() {
  563|      1|        let mut message = StompMessage::new(Ownership::Session);
  564|      1|        message.command = StompCommand::Get;
  565|      1|        message.message_type = Http;
  566|      1|        assert_eq!(message.message_type, Http);
  567|      1|        assert_eq!(message.command, StompCommand::Get);
  568|      1|    }
  569|       |
  570|      1|    #[test]
  571|      1|    fn test_extract_hdr() {
  572|      1|        let mut message = StompMessage::new(Ownership::Session);
  573|      1|        message.command = StompCommand::Ack;
  574|      1|        message.add_header("name1", "value1");
  575|      1|        message.add_header("name2", "value2");
  576|      1|        message.add_body(b"{\"resp1\": true}\n");
  577|      1|
  578|      1|        assert_eq!(message.count_headers(), 2);
  579|      1|        assert!(message.extract_header("name2").is_some());
  580|      1|        assert!(message.extract_header("name2").is_none());
  581|      1|        assert_eq!(message.count_headers(), 1);
  582|      1|    }
  583|       |}

src/parser.rs:
    1|       |//! STOMP protocol push parser, parses command and headers, not the body.
    2|       |
    3|       |/// readonly bytes pushed in from the network.
    4|       |/// Parser treats u8 as char as US-ASCII, we accept utf-8 but all protocol attributes are US-ASCII.
    5|       |/// Also reads HTTP GET to support upgrading the stream from HTTP to WebSockets
    6|       |use log::*;
    7|       |
    8|       |use crate::message::stomp_message::StompCommand;
    9|       |use crate::message::stomp_message::*;
   10|       |
   11|       |/// State of the Struct
   12|      0|#[derive(Debug, PartialEq)]
  ------------------
  | Unexecuted instantiation: _RNvXs1_NtCs9SJoeSWam1h_4romp6parserNtB5_11ParserStateNtNtCskrsM4FCwAVA_4core3cmp9PartialEq2neB7_
  ------------------
  | Unexecuted instantiation: _RNvXs1_NtCs9SJoeSWam1h_4romp6parserNtB5_11ParserStateNtNtCskrsM4FCwAVA_4core3cmp9PartialEq2eqB7_
  ------------------
   13|       |pub enum ParserState {
   14|       |    /// finished reading the whole message
   15|       |    Done(usize),
   16|       |    /// read all the headers now comes the body (value is how much data was read)
   17|       |    Message(usize),
   18|       |    /// awaiting more input
   19|       |    Again,
   20|       |    /// Bombed parsing the command (first line)
   21|       |    InvalidCommand,
   22|       |    /// bombed parsing something else
   23|       |    InvalidMessage,
   24|       |    /// flupped a buffer
   25|       |    BodyFlup,
   26|       |    /// reached max headers
   27|       |    HdrFlup,
   28|       |}
   29|       |
   30|       |/// state of the loop
   31|      7|#[derive(Debug, PartialEq)]
   32|       |enum State {
   33|       |    Start,
   34|       |    Command,
   35|       |    CommandLf,
   36|       |
   37|       |    HdrStart,
   38|       |    HdrName,
   39|       |    HdrValue,
   40|       |
   41|       |    AlmostDone,
   42|       |    MsgRead,
   43|       |}
   44|       |
   45|       |/// A push parser, externally a buffer is being filled with the STOMP message and headers as it arrive
   46|       |/// we process the data as it comes off the stream and save state between chunks of data arriving.
   47|      0|#[derive(Debug)]
   48|       |pub struct StompParser {
   49|       |    /// count of bytes read parsing the stream, can be > message len since optional whitespace is discarded.
   50|       |    bytes_read: usize,
   51|       |    state: State,
   52|       |    pub message: StompMessage,
   53|       |    cmd_start: usize,
   54|       |    request_line_start: usize,
   55|       |    hdr_name_start: usize,
   56|       |    hdr_name_end: usize,
   57|       |    hdr_value_start: usize,
   58|       |    hdr_value_end: usize,
   59|       |}
   60|       |
   61|       |impl StompParser {
   62|     12|    pub fn new(session_id: usize) -> StompParser {
   63|     12|        let mut m = StompMessage::new(Ownership::Parser);
   64|     12|        m.session_id = Some(session_id);
   65|     12|        StompParser {
   66|     12|            bytes_read: 0,
   67|     12|            state: State::Start,
   68|     12|            message: m,
   69|     12|            cmd_start: 0,
   70|     12|            request_line_start: 0,
   71|     12|            hdr_name_start: 0,
   72|     12|            hdr_name_end: 0,
   73|     12|            hdr_value_start: 0,
   74|     12|            hdr_value_end: 0,
   75|     12|        }
   76|     12|    }
   77|       |
   78|       |    /// take ownership of the parsed message
   79|      0|    pub fn take_message(&mut self, owner: Ownership) -> StompMessage {
   80|      0|        self.bytes_read = 0;
   81|      0|        self.state = State::Start;
   82|      0|        self.cmd_start = 0;
   83|      0|        self.request_line_start = 0;
   84|      0|        self.hdr_name_start = 0;
   85|      0|        self.hdr_name_end = 0;
   86|      0|        self.hdr_value_start = 0;
   87|      0|        self.hdr_value_end = 0;
   88|      0|
   89|      0|        self.message.take(owner)
   90|      0|    }
   91|       |
   92|       |    /// reset wiping the message
   93|      0|    pub fn reset(&mut self) {
   94|      0|        self.bytes_read = 0;
   95|      0|        self.state = State::Start;
   96|      0|        self.cmd_start = 0;
   97|      0|        self.request_line_start = 0;
   98|      0|        self.hdr_name_start = 0;
   99|      0|        self.hdr_name_end = 0;
  100|      0|        self.hdr_value_start = 0;
  101|      0|        self.hdr_value_end = 0;
  102|      0|
  103|      0|        let session_id = self.message.id;
  104|      0|        self.message = StompMessage::new(Ownership::Parser);
  105|      0|        self.message.session_id = Some(session_id);
  106|      0|    }
  107|       |
  108|      0|    pub fn bytes_read(&self) -> usize {
  109|      0|        self.bytes_read
  110|      0|    }
  111|       |
  112|       |    /// Required a complete rewrite from xtomp because rust does not have for loops or p++ and can not index arrays easily
  113|       |    ///
  114|       |    /// `buffer` the full buffer we are loading
  115|       |    /// `pos` position in the outer buffer (not the chunk)
  116|       |    /// `chunk` slice of input buffer we are reading
  117|     14|    pub fn push(
  118|     14|        &mut self,
  119|     14|        buffer: &[u8],
  120|     14|        mut pos: usize,
  121|     14|        chunk: &[u8],
  122|     14|    ) -> Result<ParserState, ParserState> {
  123|       |        // debug!("READ chunk '{}'", String::from_utf8_lossy(chunk));
  124|       |
  125|    223|        for c in chunk {
  126|    223|            self.bytes_read += 1;
  127|    223|            //println!("loop {}, {}", *c as char, String::from_utf8_lossy(buffer));
  128|    223|
  129|    223|            let ch: u8 = *c;
  130|    223|
  131|    223|            if ch == b'\0' && self.state != State::AlmostDone {
  132|      1|                debug!("early frame termination");
  133|      1|                return Err(ParserState::InvalidCommand);
  134|    222|            }
  135|    222|
  136|    222|            match self.state {
  137|    222|                State::Start => {
  138|     15|                    if ch == b'\n' || ch == b'\r' {
  139|      3|                        // TODO if all we got was a heart-beat we should reset buffer pos here
  140|      3|                        // continue
  141|     12|                    } else if ch < b'A' || ch > b'Z' {
  142|      2|                        return Err(ParserState::InvalidCommand);
  143|     10|                    } else {
  144|     10|                        self.cmd_start = pos;
  145|     10|                        self.state = State::Command;
  146|     10|                        self.message.command = StompCommand::Unknown;
  147|     10|                    }
  148|       |                }
  149|       |
  150|       |                State::Command => {
  151|       |                    // TODO only allow non A-Z for HTTP GET
  152|       |                    //                    if ch == b'\n' || ch == b'\r' {
  153|       |                    //
  154|       |                    //                    }
  155|       |                    //                    else if ch < b'A' || ch > b'Z' {
  156|       |                    //                        return Err(ParserState::InvalidCommand);
  157|       |                    //                    }
  158|       |
  159|     63|                    let c = self.cmd_start;
  160|     63|
  161|     63|                    // amount of command statement read
  162|     63|                    let cmd_read = pos - c;
  163|     63|                    let com = &buffer[c..c + cmd_read];
  164|     63|
  165|     63|                    //println!("Reading Command {} cmd_start={} cmd_read={}", ch, self.cmd_start, cmd_read);
  166|     63|
  167|     63|                    if cmd_read == 0 {
  168|      0|                        // continue
  169|     63|                    } else if cmd_read == 1 {
  170|     10|                        // continue
  171|     53|                    } else if cmd_read == 2 {
  172|     10|                        // continue
  173|     43|                    } else if cmd_read == 3 {
  174|       |                        //  ACK, GET
  175|     10|                        if com.eq(b"ACK") {
  176|      1|                            self.message.command = StompCommand::Ack;
  177|      9|                        }
  178|     33|                    } else if cmd_read == 4 {
  179|       |                        //  NACK, SEND
  180|      9|                        if com.eq(b"NACK") {
  181|      0|                            self.message.command = StompCommand::Nack;
  182|      9|                        }
  183|      9|                        if com.eq(b"SEND") {
  184|      1|                            self.message.command = StompCommand::Send;
  185|      8|                        }
  186|      9|                        if com.eq(b"GET ") {
  187|      1|                            self.message.command = StompCommand::Get;
  188|      1|                            self.message.message_type = MessageType::Http;
  189|      1|                            self.request_line_start = pos - 4;
  190|      8|                        }
  191|     24|                    } else if cmd_read == 5 {
  192|       |                        //  BEGIN, ABORT, ERROR, STOMP
  193|      7|                        if com.eq(b"BEGIN") {
  194|      0|                            self.message.command = StompCommand::Begin;
  195|      7|                        }
  196|      7|                        if com.eq(b"ABORT") {
  197|      0|                            self.message.command = StompCommand::Abort;
  198|      7|                        }
  199|      7|                        if com.eq(b"ERROR") {
  200|      0|                            self.message.command = StompCommand::Error;
  201|      7|                        }
  202|      7|                        if com.eq(b"STOMP") {
  203|      5|                            self.message.command = StompCommand::Stomp;
  204|      5|                        }
  205|     17|                    } else if cmd_read == 6 {
  206|       |                        //  COMMIT
  207|      2|                        if com.eq(b"COMMIT") {
  208|      0|                            self.message.command = StompCommand::Commit;
  209|      2|                        }
  210|     15|                    } else if cmd_read == 7 {
  211|       |                        //  CONNECT, MESSAGE, RECEIPT
  212|      1|                        if com.eq(b"CONNECT") {
  213|      0|                            self.message.command = StompCommand::Connect;
  214|      1|                        }
  215|      1|                        if com.eq(b"MESSAGE") {
  216|      0|                            self.message.command = StompCommand::Message;
  217|      1|                        }
  218|      1|                        if com.eq(b"RECEIPT") {
  219|      0|                            self.message.command = StompCommand::Receipt;
  220|      1|                        }
  221|     14|                    } else if cmd_read == 8 { //  none
  222|      1|                         // continue
  223|     13|                    } else if cmd_read == 9 {
  224|       |                        //  SUBSCRIBE, CONNECTED
  225|      1|                        if com.eq(b"SUBSCRIBE") {
  226|      0|                            self.message.command = StompCommand::Subscribe;
  227|      1|                        }
  228|      1|                        if com.eq(b"CONNECTED") {
  229|      0|                            self.message.command = StompCommand::Connected;
  230|      1|                        }
  231|     12|                    } else if cmd_read == 10 {
  232|       |                        //  DISCONNECT
  233|      1|                        if com.eq(b"DISCONNECT") {
  234|      0|                            self.message.command = StompCommand::Disconnect;
  235|      1|                        }
  236|     11|                    } else if cmd_read == 11 {
  237|       |                        //  UNSUBSCRIBE
  238|      1|                        if com.eq(b"UNSUBSCRIBE") {
  239|      0|                            self.message.command = StompCommand::Unsubscribe;
  240|      1|                        }
  241|     10|                    } else if cmd_read == 12 {
  242|      1|                        if StompCommand::Get == self.message.command {
  243|      1|                            // HTTP GET can be long
  244|      1|                        } else {
  245|      0|                            return Err(ParserState::InvalidCommand);
  246|       |                        }
  247|      9|                    } // end command types
  248|       |
  249|     63|                    match self.is_command_done(ch) {
  250|      8|                        Ok(state) => {
  251|      8|                            self.state = state;
  252|      8|                            self.hdr_name_start = pos;
  253|      8|                            if StompCommand::Get == self.message.command {
  254|      1|                                self.request_line_done(buffer, pos);
  255|      7|                            }
  256|       |                        }
  257|     55|                        Err(ParserState::Again) => {
  258|     54|                            // continue
  259|     54|                        }
  260|      1|                        Err(e) => {
  261|      1|                            warn!("invalid command '{}'", String::from_utf8_lossy(com));
  262|      1|                            return Err(e);
  263|       |                        }
  264|       |                    }
  265|       |                }
  266|       |
  267|       |                State::CommandLf => {
  268|      1|                    if ch == b'\r' {
  269|      0|                        // continue;
  270|      1|                    } else if ch == b'\n' {
  271|      1|                        self.state = State::HdrStart;
  272|      1|                        self.hdr_name_start = pos;
  273|      1|                        // continue;
  274|      1|                    } else {
  275|      0|                        return Err(ParserState::InvalidCommand);
  276|       |                    }
  277|       |                }
  278|       |
  279|       |                State::HdrStart => {
  280|     17|                    if ch == b' ' || ch == b'\t' || ch == b'\r' {
  281|      1|                        // continue (ignore leading whitespace)
  282|     16|                    } else if ch == b'\n' {
  283|       |                        // two LF marks end of headers
  284|      8|                        match self.message.command {
  285|      8|                            StompCommand::Message | StompCommand::Send => {
  286|       |                                // has a body, return without reading the trailing \0
  287|      1|                                self.state = State::MsgRead;
  288|      1|                                return Ok(ParserState::Message(pos + 1));
  289|       |                            }
  290|       |                            StompCommand::Get => {
  291|       |                                // has no trailing \0
  292|      1|                                self.state = State::MsgRead;
  293|      1|                                return Ok(ParserState::Message(pos + 1));
  294|       |                            }
  295|      6|                            _ => {
  296|      6|                                // wait for \0
  297|      6|                                self.state = State::AlmostDone;
  298|      6|                            }
  299|       |                        }
  300|      8|                    } else {
  301|      8|                        self.hdr_name_start = pos;
  302|      8|                        self.state = State::HdrName;
  303|      8|                    }
  304|       |                }
  305|       |
  306|       |                State::HdrName => {
  307|     56|                    if ch == b':' {
  308|      8|                        self.hdr_name_end = pos;
  309|      8|                        self.hdr_value_start = pos + 1;
  310|      8|                        self.state = State::HdrValue;
  311|     48|                    } else if ch == b'\r' || ch == b'\n' {
  312|      0|                        return Err(ParserState::InvalidMessage);
  313|     48|                    }
  314|       |                }
  315|       |
  316|       |                // TODO ignore whitespace between name and value and :
  317|       |                State::HdrValue => {
  318|     64|                    if ch == b'\r' {
  319|      2|                        self.hdr_value_end = pos;
  320|      2|                        // continue
  321|     62|                    } else if ch == b'\n' {
  322|      8|                        if self.hdr_value_end == 0 {
  323|      6|                            self.hdr_value_end = pos;
  324|      6|                        };
  325|      8|                        self.header_done(buffer);
  326|      8|
  327|      8|                        self.hdr_name_start = pos - 1;
  328|      8|                        self.state = State::HdrStart;
  329|     54|                    }
  330|       |                }
  331|       |
  332|       |                State::AlmostDone => {
  333|      6|                    if ch == b'\0' {
  334|      4|                        self.state = State::MsgRead;
  335|      4|                        debug!("DONE {} {}", pos, self.bytes_read);
  336|      4|                        return Ok(ParserState::Done(pos + 1));
  337|       |                    } else {
  338|      2|                        return Err(ParserState::InvalidCommand);
  339|       |                    }
  340|       |                }
  341|       |
  342|       |                State::MsgRead => {
  343|      0|                    panic!("unreachable in parser")
  344|       |                }
  345|       |            }
  346|       |
  347|    211|            pos = pos + 1;
  348|       |        } // end for
  349|       |
  350|      2|        Ok(ParserState::Again)
  351|     14|    }
  352|       |
  353|       |    /// return Err(ParserState::Again) until a command is read, command is the first line of the message
  354|     63|    fn is_command_done(&self, ch: u8) -> Result<State, ParserState> {
  355|     63|        if ch == b'\r' || ch == b'\n' {
  356|       |            // println!("Read a command {:?}", self.message.command);
  357|      9|            if StompCommand::Unknown == self.message.command {
  358|      1|                return Err(ParserState::InvalidCommand);
  359|      8|            }
  360|      8|            if ch == b'\r' {
  361|      1|                return Ok(State::CommandLf);
  362|      7|            }
  363|      7|            if ch == b'\n' {
  364|      7|                return Ok(State::HdrStart);
  365|      0|            }
  366|     54|        }
  367|     54|        Err(ParserState::Again)
  368|     63|    }
  369|       |
  370|       |    /// parsed a header name:value
  371|      8|    fn header_done(&mut self, buffer: &[u8]) {
  372|      8|        self.message.add_header_clone(
  373|      8|            &String::from_utf8_lossy(&buffer[self.hdr_name_start..self.hdr_name_end]),
  374|      8|            &String::from_utf8_lossy(&buffer[self.hdr_value_start..self.hdr_value_end]),
  375|      8|        );
  376|      8|
  377|      8|        self.hdr_name_start = 0;
  378|      8|        self.hdr_name_end = 0;
  379|      8|        self.hdr_value_start = 0;
  380|      8|        self.hdr_value_end = 0;
  381|      8|    }
  382|       |
  383|      1|    fn request_line_done(&mut self, buffer: &[u8], pos: usize) {
  384|      1|        //println!("read request line '{}'", String::from_utf8_lossy(&buffer[self.request_line_start..pos]) );
  385|      1|        self.message.add_header_clone(
  386|      1|            &String::from("request-line"),
  387|      1|            &String::from_utf8_lossy(&buffer[self.request_line_start..pos]),
  388|      1|        );
  389|      1|    }
  390|       |}
  391|       |
  392|       |#[cfg(test)]
  393|       |mod tests {
  394|       |    use super::*;
  395|       |
  396|      1|    #[test]
  397|      1|    fn test_happy() {
  398|      1|        let mut p = StompParser::new(0);
  399|      1|        let buffer = b"ACK\nhdr1:value1\nhdr2:value2\n\n\0";
  400|      1|        let mut pos = 0;
  401|      1|        let mut end = 10;
  402|      1|        let mut chunk = &buffer[pos..end];
  403|      1|        match p.push(buffer, pos, chunk) {
  404|      1|            Ok(ParserState::Again) => {
  405|      1|                if StompCommand::Ack == p.message.command {
  406|      1|                    println!("parsed ACK");
  407|      1|                } else {
  408|      0|                    panic!("wrong command")
  409|       |                }
  410|       |
  411|      1|                println!("p.message.command {:?}", p.message.command);
  412|      1|                println!("chunk parsed");
  413|       |            }
  414|       |            Err(_) => {
  415|      0|                println!("Parser error {:?}", p);
  416|      0|                panic!("parser error");
  417|       |            }
  418|      0|            _ => panic!("unexpected state"),
  419|       |        };
  420|       |
  421|      1|        let read = 10;
  422|      1|        pos = end;
  423|      1|        end += read;
  424|      1|        chunk = &buffer[pos..end];
  425|      1|        match p.push(buffer, pos, chunk) {
  426|      1|            Ok(ParserState::Again) => println!("chunk parsed"),
  427|      0|            Err(_) => panic!("parser error"),
  428|      0|            _ => panic!("unexpected state"),
  429|       |        };
  430|       |
  431|      1|        pos = end;
  432|      1|        end = buffer.len();
  433|      1|        chunk = &buffer[pos..end];
  434|      1|        match p.push(buffer, pos, chunk) {
  435|      1|            Ok(ParserState::Done(_)) => println!("all parsed"),
  436|      0|            Err(_) => panic!("parser error"),
  437|      0|            _ => panic!("unexpected state"),
  438|       |        };
  439|      1|        println!("Parsed Message {:?}", p.message);
  440|      1|    }
  441|       |
  442|      1|    #[test]
  443|      1|    fn test_happy_login() {
  444|      1|        let mut p = StompParser::new(0);
  445|      1|        let buffer = b"STOMP\nlogin:xtomp\npasscode:passcode\n\n\0";
  446|      1|        match p.push(buffer, 0, buffer) {
  447|      1|            Ok(ParserState::Done(38)) => {
  448|      1|                if StompCommand::Stomp == p.message.command {
  449|      1|                } else {
  450|      0|                    panic!("wrong command");
  451|       |                }
  452|      1|                match p.message.get_header("login") {
  453|      1|                    Some(value) => assert!(value.eq("xtomp")),
  454|      0|                    None => panic!("missing header"),
  455|       |                }
  456|      1|                match p.message.get_header("passcode") {
  457|      1|                    Some(value) => assert!(value.eq("passcode")),
  458|      0|                    None => panic!("missing header"),
  459|       |                }
  460|       |            }
  461|       |            Ok(ParserState::Done(_)) => {
  462|      0|                panic!("reporting wrong length")
  463|       |            }
  464|       |            Err(_) => {
  465|      0|                println!("Parser error {:?}", p);
  466|      0|                panic!("parser error")
  467|       |            }
  468|      0|            _ => panic!("unexpected state"),
  469|       |        };
  470|      1|    }
  471|       |
  472|      1|    #[test]
  473|      1|    fn test_happy_body() {
  474|      1|        let mut p = StompParser::new(0);
  475|      1|        let buffer = b"SEND\ndestination:memtop-a\n\nsome text follows\0";
  476|      1|        match p.push(buffer, 0, buffer) {
  477|      1|            Ok(ParserState::Message(read)) => {
  478|      1|                assert_eq!(27, read);
  479|      1|                if StompCommand::Send == p.message.command {
  480|      1|                } else {
  481|      0|                    panic!("wrong command");
  482|       |                }
  483|      1|                match p.message.get_header("destination") {
  484|      1|                    Some(value) => assert!(value.eq("memtop-a")),
  485|      0|                    None => panic!("missing header"),
  486|       |                }
  487|      1|                if State::MsgRead == p.state {
  488|      1|                } else {
  489|      0|                    panic!("wrong parser state");
  490|       |                }
  491|       |            }
  492|       |            Err(_) => {
  493|      0|                println!("Parser error {:?}", p);
  494|      0|                panic!("parser error")
  495|       |            }
  496|      0|            _ => panic!("unexpected state"),
  497|       |        };
  498|      1|    }
  499|       |
  500|      1|    #[test]
  501|      1|    fn test_telnet_line_endings_login() {
  502|      1|        let mut p = StompParser::new(0);
  503|      1|        let buffer = b"STOMP\r\nlogin:xtomp\r\npasscode:passcode\r\n\r\n\0";
  504|      1|        match p.push(buffer, 0, buffer) {
  505|      1|            Ok(ParserState::Done(_)) => {
  506|      1|                if StompCommand::Stomp == p.message.command {
  507|      1|                } else {
  508|      0|                    panic!("wrong command");
  509|       |                }
  510|      1|                match p.message.get_header("login") {
  511|      1|                    Some(value) => assert!(value.eq("xtomp")),
  512|      0|                    None => panic!("missing header"),
  513|       |                }
  514|      1|                match p.message.get_header("passcode") {
  515|      1|                    Some(value) => assert!(value.eq("passcode")),
  516|      0|                    None => panic!("missing header"),
  517|       |                }
  518|       |            }
  519|       |            Err(_) => {
  520|      0|                println!("Parser error {:?}", p);
  521|      0|                panic!("parser error")
  522|       |            }
  523|      0|            _ => panic!("unexpected state"),
  524|       |        };
  525|      1|    }
  526|       |
  527|      1|    #[test]
  528|      1|    fn test_invalid_command() {
  529|      1|        let buffer = b"WIBBLE\n\n\0";
  530|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  531|      1|            // expected
  532|      1|        } else {
  533|      0|            panic!("unexpected state")
  534|       |        }
  535|      1|    }
  536|       |
  537|      1|    #[test]
  538|      1|    fn test_non_text_command() {
  539|      1|        let buffer = b"*uck*you\n\n\0";
  540|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  541|      1|            // expected
  542|      1|        } else {
  543|      0|            panic!("unexpected state")
  544|       |        }
  545|      1|    }
  546|       |
  547|      1|    #[test]
  548|      1|    fn test_wrong_case_command() {
  549|      1|        let buffer = b"stomp\n\n\0";
  550|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  551|      1|            // expected
  552|      1|        } else {
  553|      0|            panic!("unexpected state")
  554|       |        }
  555|      1|    }
  556|       |
  557|      1|    #[test]
  558|      1|    fn test_heart_beats() {
  559|      1|        let mut p = StompParser::new(0);
  560|      1|        // leading \n ignored
  561|      1|        let buffer = b"\n\n\nSTOMP\n\n\0";
  562|      1|        if let Ok(_) = p.push(buffer, 0, buffer) {
  563|      1|            assert_eq!(StompCommand::Stomp, p.message.command);
  564|       |        } else {
  565|      0|            panic!("unexpected state")
  566|       |        }
  567|      1|    }
  568|       |
  569|      1|    #[test]
  570|      1|    fn test_early_zero_termination_in_cmd() {
  571|      1|        let buffer = b"STOMP\0";
  572|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  573|      1|            // expected
  574|      1|        } else {
  575|      0|            panic!("unexpected state")
  576|       |        }
  577|      1|    }
  578|       |
  579|      1|    #[test]
  580|      1|    fn test_early_zero_termination_in_hdr_val() {
  581|      1|        let buffer = b"STOMP\n\nhdr:value\0";
  582|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  583|      1|            // expected
  584|      1|        } else {
  585|      0|            panic!("unexpected state")
  586|       |        }
  587|      1|    }
  588|       |
  589|      1|    #[test]
  590|      1|    fn test_early_zero_termination_in_hdr_name() {
  591|      1|        let buffer = b"STOMP\n\nhdr:value\n\0";
  592|      1|        if let Err(ParserState::InvalidCommand) = StompParser::new(0).push(buffer, 0, buffer) {
  593|      1|            // expected
  594|      1|        } else {
  595|      0|            panic!("unexpected state")
  596|       |        }
  597|      1|    }
  598|       |
  599|      1|    #[test]
  600|      1|    fn test_http_get() {
  601|      1|        let mut p = StompParser::new(0);
  602|      1|        let buffer = b"GET /foo/baa HTTP/1.1\ndestination:memtop-a\n\n";
  603|      1|        match p.push(buffer, 0, buffer) {
  604|      1|            Ok(ParserState::Message(read)) => {
  605|      1|                assert_eq!(44, read);
  606|      1|                if StompCommand::Get == p.message.command {
  607|      1|                } else {
  608|      0|                    panic!("wrong command");
  609|       |                }
  610|      1|                match p.message.get_header("destination") {
  611|      1|                    Some(value) => assert!(value.eq("memtop-a")),
  612|      0|                    None => panic!("missing header"),
  613|       |                }
  614|      1|                if State::MsgRead == p.state {
  615|      1|                } else {
  616|      0|                    panic!("wrong parser state");
  617|       |                }
  618|       |            }
  619|       |            Err(_) => {
  620|      0|                println!("Parser error {:?}", p);
  621|      0|                panic!("parser error")
  622|       |            }
  623|      0|            _ => panic!("unexpected state"),
  624|       |        };
  625|      1|    }
  626|       |}

src/persist/mlog.rs:
    1|       |//! Not the most efficient possible message logger, uses synchronous IO, this is acceptable in
    2|       |//! many situations in Linux since `write()` is buffered by the kernel and returns quite quick.
    3|       |//! There is no guarantee that a message is written when the `store()` method returns.
    4|       |
    5|       |use std::fs::{File, OpenOptions};
    6|       |use std::io;
    7|       |use std::io::Write;
    8|       |
    9|       |use chrono::SecondsFormat;
   10|       |
   11|       |use crate::message::stomp_message::StompMessage;
   12|       |use std::sync::atomic::{AtomicBool, Ordering};
   13|       |use std::sync::Mutex;
   14|       |
   15|       |// message logging
   16|       |
   17|       |pub trait MLog {
   18|       |    fn store(&self, user: &str, message: &StompMessage) -> io::Result<usize>;
   19|       |}
   20|       |
   21|       |pub struct SyncMLog {
   22|       |    file: Mutex<File>,
   23|       |    err: AtomicBool,
   24|       |}
   25|       |
   26|       |impl SyncMLog {
   27|      1|    pub fn new(file_name: &str) -> Result<SyncMLog, i32> {
   28|      1|        return match OpenOptions::new()
   29|      1|            .append(true)
   30|      1|            .write(true)
   31|      1|            .create(true)
   32|      1|            .open(file_name)
   33|       |        {
   34|      1|            Ok(file) => Ok(SyncMLog {
   35|      1|                file: Mutex::new(file),
   36|      1|                err: AtomicBool::from(false),
   37|      1|            }),
   38|      0|            _ => Err(-1),
   39|       |        };
   40|      1|    }
   41|       |}
   42|       |
   43|       |impl MLog for SyncMLog {
   44|      1|    fn store(&self, user: &str, message: &StompMessage) -> io::Result<usize> {
   45|      1|        if self.err.load(Ordering::Relaxed) {
   46|      0|            return Err(io::Error::from(io::ErrorKind::Other));
   47|      1|        }
   48|      1|        let header = format!(
   49|      1|            "MSG {} {} {} {} {}\n",
   50|      1|            message.id,
   51|      1|            message.header_len(),
   52|      1|            message.body_len(),
   53|      1|            message
   54|      1|                .timestamp
   55|      1|                .to_rfc3339_opts(SecondsFormat::Millis, true),
   56|      1|            user
   57|      1|        );
   58|      1|
   59|      1|        // Writing to a memory buffer and a single `write()` syscall might be faster.
   60|      1|        let mut file = self.file.lock().unwrap();
   61|      1|        if let Err(e) = file.write(header.as_bytes()) {
   62|      0|            self.err.store(true, Ordering::Relaxed);
   63|      0|            return Err(e);
   64|      1|        }
   65|       |        // causes a copy
   66|      1|        if let Err(e) = file.write(message.hdrs_as_string().as_bytes()) {
   67|      0|            self.err.store(true, Ordering::Relaxed);
   68|      0|            return Err(e);
   69|      1|        }
   70|       |        // causes a copy
   71|      1|        if let Err(e) = file.write(message.combine().as_slice()) {
   72|      0|            self.err.store(true, Ordering::Relaxed);
   73|      0|            return Err(e);
   74|      1|        }
   75|      1|        if let Err(e) = file.write(&[0 as u8, b'\n']) {
   76|      0|            self.err.store(true, Ordering::Relaxed);
   77|      0|            return Err(e);
   78|      1|        }
   79|      1|
   80|      1|        Ok(0)
   81|      1|    }
   82|       |}
   83|       |
   84|       |#[cfg(test)]
   85|       |mod tests {
   86|       |    use super::*;
   87|       |
   88|      1|    #[test]
   89|      1|    fn test_happy() {
   90|      1|        let mut message = StompMessage::new_send(b"some data", 0);
   91|      1|        message.add_header("some", "header");
   92|      1|
   93|      1|        let mlog = SyncMLog::new("/tmp/test.mlog").unwrap();
   94|      1|        mlog.store("teknopaul", &message)
   95|      1|            .expect("mlog write failed");
   96|      1|    }
   97|       |}

src/session/ecg.rs:
    1|       |//! `heart-beat` functions.
    2|       |
    3|       |use std::sync::{Arc, RwLock};
    4|       |
    5|       |use log::*;
    6|       |
    7|       |use crate::init::CONFIG;
    8|       |
    9|       |use crate::message::stomp_message::{Header, StompMessage};
   10|       |use crate::session::stomp_session::StompSession;
   11|       |
   12|       |// contains code for heart-beats, (keep-alive pings)
   13|       |
   14|      0|pub fn get_request_header(message: &StompMessage) -> Option<&Header> {
   15|      0|    for hdr in message.headers() {
   16|      0|        if hdr.name.eq("heart-beat") {
   17|      0|            return Some(hdr);
   18|      0|        }
   19|       |    }
   20|      0|    None
   21|      0|}
   22|       |
   23|       |/// get the header value to add to a CONNECTED reply
   24|      0|pub fn get_response_header(session: &Arc<RwLock<StompSession>>) -> Header {
   25|      0|    let session = session.read().unwrap();
   26|      0|    Header {
   27|      0|        name: String::from("heart-beat"),
   28|      0|        value: format!("{},{}", session.heart_beat_write, session.heart_beat_read),
   29|      0|    }
   30|      0|}
   31|       |
   32|       |/// parse header and update the StompSession with expected heart-beat config
   33|      7|pub fn parse_header(hdr: &Header, session: &Arc<RwLock<StompSession>>) -> Result<(), &'static str> {
   34|      7|    // Rust is sooo ugly sometimes this parses "x,y"
   35|      7|
   36|      7|    let comma_idx = (*hdr.value).find(',');
   37|      7|    match comma_idx {
   38|      7|        Some(comma_idx) => {
   39|      7|            let (cx, cy) = (*hdr.value).split_at(comma_idx);
   40|      7|            if cy.len() < 2 {
   41|      1|                return Err("ecg syntax");
   42|      6|            }
   43|      6|            // atoi(cx, cy)
   44|      6|            match (cx.parse::<u32>(), cy[1..].parse::<u32>()) {
   45|      5|                (Ok(cx), Ok(cy)) => {
   46|      5|                    let mut s = session.write().unwrap();
   47|      5|
   48|      5|                    if cx == 0 {
   49|      1|                        s.heart_beat_read = 0;
   50|      4|                    } else if cx > CONFIG.heart_beat_read {
   51|       |                        // error client can only guarantee beats every cx but we need beats every  CONFIG.heart_beat_read
   52|      1|                        debug!("ecg incompat");
   53|      1|                        return Err("ecg incompat");
   54|      3|                    } else {
   55|      3|                        s.heart_beat_read = CONFIG.heart_beat_read;
   56|      3|                    }
   57|       |
   58|      4|                    if cy == 0 || CONFIG.heart_beat_write_max == 0 {
   59|      1|                        s.heart_beat_write = 0;
   60|      3|                    } else if cy < CONFIG.heart_beat_write_min {
   61|      1|                        debug!("ecg incompat");
   62|      1|                        return Err("ecg incompat");
   63|      2|                    } else if cy > CONFIG.heart_beat_write_max {
   64|      1|                        s.heart_beat_write = CONFIG.heart_beat_write_max;
   65|      1|                    } else {
   66|      1|                        s.heart_beat_write = cy;
   67|      1|                    }
   68|       |
   69|      3|                    return Ok(());
   70|       |                }
   71|      1|                _ => return Err("ecg syntax"),
   72|       |            }
   73|       |        }
   74|       |        _ => {
   75|      0|            return Err("ecg syntax");
   76|       |        }
   77|       |    }
   78|      7|}
   79|       |
   80|       |#[cfg(test)]
   81|       |mod tests {
   82|       |
   83|       |    use super::*;
   84|       |
   85|      1|    #[test]
   86|      1|    fn test_parse_header() {
   87|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
   88|      1|        let hdr = Header::new("heart-beat", "60000,60000");
   89|      1|        match parse_header(&hdr, &session) {
   90|      1|            Ok(()) => {}
   91|      0|            Err(e) => panic!("parse failed '{}'", e),
   92|       |        }
   93|       |
   94|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
   95|      1|        let hdr = Header::new("heart-beat", "60000,");
   96|      1|        match parse_header(&hdr, &session) {
   97|      1|            Ok(()) => panic!("expected parse fail"),
   98|      1|            Err(_e) => {}
   99|      1|        }
  100|      1|
  101|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
  102|      1|        let hdr = Header::new("heart-beat", ",60000");
  103|      1|        match parse_header(&hdr, &session) {
  104|      1|            Ok(()) => panic!("expected parse fail"),
  105|      1|            Err(_e) => {}
  106|      1|        }
  107|      1|
  108|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
  109|      1|        let hdr = Header::new("heart-beat", "60000,600000");
  110|      1|        match parse_header(&hdr, &session) {
  111|      1|            Ok(()) => {
  112|      1|                let session = session.read().unwrap();
  113|      1|                assert_eq!(session.heart_beat_read, 120000); // N.B. heart_beat_read set to server config, client will send us more often than this, thats OK
  114|      1|                assert_eq!(session.heart_beat_write, CONFIG.heart_beat_write_max);
  115|       |            }
  116|      0|            Err(_e) => {}
  117|       |        }
  118|       |
  119|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
  120|      1|        let hdr = Header::new("heart-beat", "60000,6000");
  121|      1|        match parse_header(&hdr, &session) {
  122|      1|            Ok(()) => {
  123|      0|                let session = session.read().unwrap();
  124|      0|                assert_eq!(session.heart_beat_read, 120000);
  125|      0|                assert_eq!(session.heart_beat_write, CONFIG.heart_beat_write_min);
  126|       |            }
  127|      1|            Err(_e) => {}
  128|       |        }
  129|       |
  130|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
  131|      1|        let hdr = Header::new("heart-beat", "0,0");
  132|      1|        match parse_header(&hdr, &session) {
  133|      1|            Ok(()) => {
  134|      1|                let session = session.read().unwrap();
  135|      1|                assert_eq!(session.heart_beat_read, 0);
  136|      1|                assert_eq!(session.heart_beat_write, 0);
  137|       |            }
  138|      0|            Err(_e) => {}
  139|       |        }
  140|      1|    }
  141|       |
  142|      1|    #[test]
  143|      1|    fn test_parse_header_read_range() {
  144|      1|        let session = Arc::new(RwLock::new(StompSession::new()));
  145|      1|        let hdr = Header::new("heart-beat", "120001,60000");
  146|      1|        match parse_header(&hdr, &session) {
  147|      1|            Ok(()) => panic!("expect read out of range"),
  148|      1|            Err(_e) => {}
  149|      1|        }
  150|      1|    }
  151|       |}

src/session/filter.rs:
    1|       |//! Code for filtering messages, (not workflow filters). `xtomp` stores messages pre-filtered which is faster but only supports simple filters
    2|       |//! `romp` runs the the filters on each message for each subscription which enables more elaborate filters.
    3|       |//! `romp` requires that the header names follow `sanename.org` package name restrictions
    4|       |
    5|       |use crate::message::stomp_message::StompMessage;
    6|       |
    7|      7|#[derive(Debug, PartialEq)]
    8|       |enum Operator {
    9|       |    Equal,
   10|       |    NotEqual,
   11|       |    NumericEquals,
   12|       |    GreaterThan,
   13|       |    LessThan,
   14|       |}
   15|       |
   16|      0|#[derive(Debug)]
   17|       |pub struct Filter {
   18|       |    hdr: String,
   19|       |    op: Operator,
   20|       |    value: String,
   21|       |}
   22|       |
   23|       |impl Filter {
   24|       |    /// Parse "name=value" no trim()ing of the input occurs
   25|       |    // Parsing a string in rust is horrible always.
   26|       |    // TODO This is shitty code fix it so it does no allocations and is not so fugly
   27|      4|    pub fn parse(definition: &String) -> Result<Filter, ()> {
   28|      4|        let mut state = 0;
   29|      4|        let mut hdr = String::new();
   30|      4|        let mut operator = String::new();
   31|      4|        let mut value = String::new();
   32|     21|        for c in definition.chars() {
   33|     21|            match state {
   34|       |                0 => {
   35|     16|                    if (c >= 'a' && c <= 'z') || c == '-' {
   36|     12|                        hdr.push(c);
   37|     12|                    } else {
   38|      4|                        operator.push(c);
   39|      4|                        state += 1;
   40|      4|                    }
   41|       |                }
   42|       |                1 => {
   43|      5|                    if c == '=' || c == '!' || c == '>' || c == '<' {
   44|      1|                        operator.push(c);
   45|      4|                    } else {
   46|      4|                        value.push(c);
   47|      4|                        state += 1;
   48|      4|                    }
   49|       |                }
   50|      0|                _ => {
   51|      0|                    value.push(c);
   52|      0|                }
   53|       |            }
   54|       |        }
   55|       |        let op;
   56|      4|        match operator.as_str() {
   57|      4|            "=" => op = Operator::Equal,
   58|      3|            "==" => op = Operator::NumericEquals,
   59|      2|            "!=" => op = Operator::NotEqual,
   60|      2|            ">" => op = Operator::GreaterThan,
   61|      1|            "<" => op = Operator::LessThan,
   62|      0|            _ => op = Operator::Equal,
   63|       |        }
   64|      4|        if op == Operator::GreaterThan || op == Operator::GreaterThan {
   65|      1|            if let Err(_) = value.parse::<i64>() {
   66|      0|                return Err(());
   67|      1|            }
   68|      3|        }
   69|      4|        Ok(Filter { hdr, op, value })
   70|      4|    }
   71|       |
   72|       |    pub fn matches_message(&self, message: &StompMessage) -> bool {
   73|      8|        if let Some(to_match) = message.get_header(&self.hdr) {
   74|      8|            match self.op {
   75|      8|                Operator::Equal => return to_match == self.value.as_str(),
   76|      0|                Operator::NotEqual => return to_match != self.value.as_str(),
   77|       |                Operator::NumericEquals => {
   78|      2|                    if let Ok(int) = to_match.parse::<i64>() {
   79|      2|                        return int == self.value.parse::<i64>().unwrap();
   80|      0|                    }
   81|      0|                    return false;
   82|       |                }
   83|       |                Operator::LessThan => {
   84|      2|                    if let Ok(int) = to_match.parse::<i64>() {
   85|      2|                        return int < self.value.parse::<i64>().unwrap();
   86|      0|                    }
   87|      0|                    return false;
   88|       |                }
   89|       |                Operator::GreaterThan => {
   90|      2|                    if let Ok(int) = to_match.parse::<i64>() {
   91|      2|                        return int > self.value.parse::<i64>().unwrap();
   92|      0|                    }
   93|      0|                    return false;
   94|       |                }
   95|       |            };
   96|      0|        }
   97|      0|        false
   98|      8|    }
   99|       |}
  100|       |
  101|       |#[cfg(test)]
  102|       |mod tests {
  103|       |    use super::*;
  104|       |    use crate::message::stomp_message::Ownership;
  105|       |
  106|      1|    #[test]
  107|      1|    pub fn test_string_equal() {
  108|      1|        let filter = Filter::parse(&String::from("grp=1")).unwrap();
  109|      1|
  110|      1|        let mut message = StompMessage::new(Ownership::Destination);
  111|      1|        message.add_header("grp", "1");
  112|      1|
  113|      1|        if !filter.matches_message(&message) {
  114|      0|            panic!("grp=1");
  115|      1|        }
  116|      1|
  117|      1|        let mut message = StompMessage::new(Ownership::Destination);
  118|      1|        message.add_header("grp", "2");
  119|      1|
  120|      1|        if filter.matches_message(&message) {
  121|      0|            panic!("grp=2");
  122|      1|        }
  123|      1|    }
  124|       |
  125|      1|    #[test]
  126|      1|    pub fn test_numeric_equal() {
  127|      1|        let filter = Filter::parse(&String::from("grp==1")).unwrap();
  128|      1|
  129|      1|        let mut message = StompMessage::new(Ownership::Destination);
  130|      1|        message.add_header("grp", "1");
  131|      1|
  132|      1|        if !filter.matches_message(&message) {
  133|      0|            panic!("grp==1");
  134|      1|        }
  135|      1|
  136|      1|        let mut message = StompMessage::new(Ownership::Destination);
  137|      1|        message.add_header("grp", "2");
  138|      1|
  139|      1|        if filter.matches_message(&message) {
  140|      0|            panic!("grp==2");
  141|      1|        }
  142|      1|    }
  143|       |
  144|      1|    #[test]
  145|      1|    pub fn test_numeric_gt() {
  146|      1|        let filter = Filter::parse(&String::from("grp>1")).unwrap();
  147|      1|
  148|      1|        let mut message = StompMessage::new(Ownership::Destination);
  149|      1|        message.add_header("grp", "2");
  150|      1|
  151|      1|        if !filter.matches_message(&message) {
  152|      0|            panic!("grp>2");
  153|      1|        }
  154|      1|
  155|      1|        let mut message = StompMessage::new(Ownership::Destination);
  156|      1|        message.add_header("grp", "1");
  157|      1|
  158|      1|        if filter.matches_message(&message) {
  159|      0|            panic!("grp>1");
  160|      1|        }
  161|      1|    }
  162|       |
  163|      1|    #[test]
  164|      1|    pub fn test_numeric_lt() {
  165|      1|        let filter = Filter::parse(&String::from("grp<1")).unwrap();
  166|      1|
  167|      1|        let mut message = StompMessage::new(Ownership::Destination);
  168|      1|        message.add_header("grp", "0");
  169|      1|
  170|      1|        if !filter.matches_message(&message) {
  171|      0|            panic!("grp<0");
  172|      1|        }
  173|      1|
  174|      1|        let mut message = StompMessage::new(Ownership::Destination);
  175|      1|        message.add_header("grp", "1");
  176|      1|
  177|      1|        if filter.matches_message(&message) {
  178|      0|            panic!("grp<1");
  179|      1|        }
  180|      1|    }
  181|       |}

src/session/mq.rs:
    1|       |//! Session specific message queue of StompMessages that are pending write.
    2|       |
    3|       |use std::sync::Arc;
    4|       |
    5|       |use log::*;
    6|       |use tokio::prelude::task::Task;
    7|       |use tokio::prelude::Async;
    8|       |
    9|       |use crate::message::stomp_message::{Header, StompMessage};
   10|       |
   11|       |/// Outgoing queue of messages the session is going to send
   12|       |/// TODO flup the mq so slow consumers cant hog memory
   13|       |
   14|       |/// A single message and headers specific to this session, e.g. subscription it came from, or session id
   15|       |pub struct SessionMessage {
   16|       |    pub message: Arc<StompMessage>,
   17|       |    // TODO could use &'static str for name in these headers to avoid some unnecessary allocations
   18|       |    pub headers: Vec<Header>,
   19|       |}
   20|       |impl SessionMessage {
   21|      8|    pub fn session_hdr_len(&self) -> usize {
   22|      8|        let mut len: usize = 0;
   23|      2|        for h in &self.headers {
   24|      2|            len += h.len();
   25|      2|            len += 1;
   26|      2|        }
   27|      8|        len
   28|      8|    }
   29|       |}
   30|       |
   31|       |pub struct Mq {
   32|       |    // TODO nasty that all messages are behind Arc when some can be owned and others could be 'static strings
   33|       |    q: Vec<SessionMessage>,
   34|       |    task: Option<Task>,
   35|       |    // close when q is empty
   36|       |    drain: bool,
   37|       |    // empty and closed
   38|       |    close: bool,
   39|       |}
   40|       |
   41|       |impl Mq {
   42|      8|    pub fn new() -> Mq {
   43|      8|        Mq {
   44|      8|            q: vec![],
   45|      8|            task: None,
   46|      8|            drain: false,
   47|      8|            close: false,
   48|      8|        }
   49|      8|    }
   50|       |
   51|      0|    pub fn set_task(&mut self, task: Task) {
   52|      0|        self.task = Some(task);
   53|      0|    }
   54|       |
   55|      0|    pub fn push(&mut self, message: Arc<StompMessage>, headers: Vec<Header>) -> usize {
   56|      0|        if self.drain | self.close {
   57|      0|            debug!("dropping message, session closing");
   58|      0|            return self.q.len();
   59|      0|        }
   60|      0|
   61|      0|        debug!("mq pushed {:?} len={}", message.command, self.q.len());
   62|       |
   63|      0|        self.q.push(SessionMessage { message, headers });
   64|      0|
   65|      0|        match &self.task {
   66|      0|            Some(task) => {
   67|      0|                debug!("mq notified");
   68|      0|                task.notify();
   69|       |            }
   70|       |            _ => {
   71|      0|                debug!("no-one to notify");
   72|       |            }
   73|       |        }
   74|       |
   75|      0|        self.q.len()
   76|      0|    }
   77|       |
   78|       |    /// pops a message and transfers ownership
   79|      0|    pub fn next(&mut self) -> Option<SessionMessage> {
   80|      0|        if self.q.len() > 0 {
   81|      0|            return Some(self.q.remove(0));
   82|      0|        }
   83|      0|        None
   84|      0|    }
   85|       |
   86|      0|    pub fn len(&self) -> usize {
   87|      0|        self.q.len()
   88|      0|    }
   89|       |
   90|      0|    pub fn close(&mut self) {
   91|      0|        self.q.clear();
   92|      0|        self.close = true;
   93|      0|        self.notify();
   94|      0|        self.task = None;
   95|      0|    }
   96|       |
   97|      0|    pub fn is_closed(&self) -> bool {
   98|      0|        self.close
   99|      0|    }
  100|       |
  101|       |    /// Tell the MQ to stop accepting new messages and carry on writing any existing ones.
  102|       |    /// When there are none left shutdown
  103|      0|    pub fn drain(&mut self) -> Result<(), usize> {
  104|      0|        self.drain = true;
  105|      0|        self.notify();
  106|      0|
  107|      0|        match self.q.len() {
  108|       |            0 => {
  109|      0|                self.close = true;
  110|      0|                self.task = None;
  111|      0|                Ok(())
  112|       |            }
  113|      0|            len => Err(len),
  114|       |        }
  115|      0|    }
  116|       |
  117|      0|    pub fn notify(&self) {
  118|      0|        match &self.task {
  119|      0|            Some(task) => task.notify(),
  120|      0|            _ => {}
  121|       |        }
  122|      0|    }
  123|       |
  124|       |    // not a real future only called by Writer via session
  125|       |    // N.B. does not require a write lock
  126|      0|    pub fn poll(&self) -> Result<Async<()>, ()> {
  127|      0|        debug!("mq polled with {} messages on the q", self.q.len());
  128|      0|        if self.close {
  129|      0|            if self.len() > 0 {
  130|      0|                warn!("close with messages on the q");
  131|      0|            }
  132|      0|            debug!("mq closed");
  133|      0|            return Err(());
  134|      0|        }
  135|      0|        match self.q.len() {
  136|       |            0 => {
  137|      0|                if self.drain {
  138|      0|                    debug!("mq closed");
  139|      0|                    return Err(());
  140|      0|                }
  141|      0|                Ok(Async::NotReady)
  142|       |            }
  143|       |            _ => {
  144|      0|                debug!("mq ready");
  145|      0|                Ok(Async::Ready(()))
  146|       |            }
  147|       |        }
  148|      0|    }
  149|       |}

src/session/reader.rs:
    1|       |use std::sync::{Arc, RwLock};
    2|       |
    3|       |use log::Level::Debug;
    4|       |use log::*;
    5|       |
    6|       |use tokio::io::ReadHalf;
    7|       |use tokio::net::TcpStream;
    8|       |use tokio::prelude::task::Task;
    9|       |use tokio::prelude::*;
   10|       |
   11|       |use crate::body_parser::fixed_length::FixedLengthBodyParser;
   12|       |use crate::body_parser::text::TextBodyParser;
   13|       |use crate::body_parser::BodyParser;
   14|       |use crate::body_parser::ParserState::BodyFlup;
   15|       |use crate::errors::ClientError;
   16|       |use crate::init::CONFIG;
   17|       |use crate::message::stomp_message::Ownership;
   18|       |use crate::message::stomp_message::StompCommand;
   19|       |use crate::parser::{ParserState, StompParser};
   20|       |use crate::session::stomp_session::{StompSession, FLAG_WEB_SOCKETS};
   21|       |use crate::web_socket::ws_demunge::{ws_demunge, WsState};
   22|       |use crate::workflow::router;
   23|       |
   24|       |/// responsible for reading from the Tcp connection's ReadHalf
   25|       |
   26|       |enum ReaderMode {
   27|       |    ReadingCommand,
   28|       |    ReadingTextBody,
   29|       |    ReadingBinaryBody,
   30|       |}
   31|       |
   32|       |pub struct Reader {
   33|       |    session: Arc<RwLock<StompSession>>,
   34|       |    session_id: usize,
   35|       |    read_half: ReadHalf<TcpStream>,
   36|       |    buf: Box<[u8]>,
   37|       |    // how far into buf we have analyzed
   38|       |    pos: usize,
   39|       |    // how much data has been pushed to buf
   40|       |    end: usize,
   41|       |    read_done: bool,
   42|       |    web_socket: bool,
   43|       |    ws: WsState,
   44|       |    // q of frame lengths so we can validate frame length == STOMP msg length
   45|       |    frame_lengths: Box<Vec<usize>>,
   46|       |    mode: ReaderMode,
   47|       |    parser: StompParser,
   48|       |    // both have trait BodyParser but this does not work with stack references only for boxed vars (unknown size)
   49|       |    fixed_length_body_parser: FixedLengthBodyParser,
   50|       |    text_body_parser: TextBodyParser,
   51|       |}
   52|       |
   53|       |impl Reader {
   54|      0|    pub fn new(
   55|      0|        session: Arc<RwLock<StompSession>>,
   56|      0|        session_id: usize,
   57|      0|        read_half: ReadHalf<TcpStream>,
   58|      0|    ) -> Reader {
   59|      0|        Reader {
   60|      0|            session,
   61|      0|            session_id,
   62|      0|            read_half,
   63|      0|            buf: vec![0 as u8; CONFIG.request_client_buffer].into_boxed_slice(),
   64|      0|            pos: 0,
   65|      0|            end: 0,
   66|      0|            read_done: false,
   67|      0|            web_socket: false,
   68|      0|            ws: WsState::new(),
   69|      0|            frame_lengths: Box::new(vec![]),
   70|      0|            mode: ReaderMode::ReadingCommand,
   71|      0|            parser: StompParser::new(session_id),
   72|      0|            fixed_length_body_parser: FixedLengthBodyParser::new(),
   73|      0|            text_body_parser: TextBodyParser::new(CONFIG.max_message_size),
   74|      0|        }
   75|      0|    }
   76|       |
   77|       |    /// called when Stomp command and headers have been read, unless its a SEND, \0 has also been read.
   78|      0|    fn handle_command_received(&mut self) -> Result<(), ParserState> {
   79|      0|        let message = &self.parser.message;
   80|      0|        if let Err(ps) = self.assert_hdr_lengths() {
   81|      0|            return Err(ps);
   82|      0|        }
   83|      0|        match &message.command {
   84|      0|            StompCommand::Send | StompCommand::Message => {
   85|      0|                debug!("headers read, reading body");
   86|      0|                self.setup_body_parser();
   87|       |            }
   88|       |            // we read a trailing \0
   89|       |            _ => {
   90|      0|                if !self.assert_ws_frame_len() {
   91|      0|                    return Err(ParserState::InvalidCommand);
   92|      0|                }
   93|      0|                self.route_message();
   94|       |            }
   95|       |        }
   96|      0|        self.realign_buffer();
   97|      0|        if self.end > 0 {
   98|      0|            return self.process_buffer_portion();
   99|       |        } else {
  100|      0|            return Ok(());
  101|       |        }
  102|      0|    }
  103|       |
  104|       |    /// Called when a message body has been completely read to memory
  105|      0|    fn handle_message_received(&mut self) -> Result<(), ParserState> {
  106|      0|        if !self.assert_ws_frame_len() {
  107|      0|            return Err(ParserState::InvalidCommand);
  108|      0|        }
  109|      0|        debug!("body read");
  110|      0|        self.realign_buffer();
  111|      0|
  112|      0|        self.route_message();
  113|      0|
  114|      0|        self.mode = ReaderMode::ReadingCommand;
  115|      0|
  116|      0|        if self.end > 0 {
  117|      0|            return self.process_buffer_portion();
  118|       |        } else {
  119|      0|            return Ok(());
  120|       |        }
  121|      0|    }
  122|       |
  123|      0|    fn assert_hdr_lengths(&self) -> Result<(), ParserState> {
  124|      0|        let message = &self.parser.message;
  125|      0|        // TODO be good to exclude required headers from this count
  126|      0|        // 8 is bear minimum for Http Upgrade, could separate counts for STOMP and Http
  127|      0|        if message.count_headers() > CONFIG.max_headers {
  128|      0|            debug!("hdr flup count");
  129|      0|            return Err(ParserState::HdrFlup);
  130|      0|        }
  131|      0|        for hdr in message.headers() {
  132|      0|            if hdr.name.len() + 1 + hdr.value.len() > CONFIG.max_header_len {
  133|      0|                debug!("hdr flup len");
  134|      0|                return Err(ParserState::HdrFlup);
  135|      0|            }
  136|       |        }
  137|      0|        Ok(())
  138|      0|    }
  139|       |
  140|       |    /// check the amount we have read parsing the data is the ws.frame_len
  141|       |    /// this allows failing fast, rather than trying to parse message data as websockets binary frame information.
  142|       |    /// This could be optional so we allow multiple STOMP messages in a single ws frame, error detection is harder if we do
  143|      0|    fn assert_ws_frame_len(&mut self) -> bool {
  144|      0|        if self.web_socket {
  145|       |            let actual;
  146|      0|            if self.fixed_length_body_parser.expected_len() > 0 {
  147|      0|                actual =
  148|      0|                    self.parser.bytes_read() + self.fixed_length_body_parser.expected_len() + 1;
  149|      0|            } else if self.text_body_parser.bytes_read() > 0 {
  150|      0|                actual = self.parser.bytes_read() + self.text_body_parser.bytes_read();
  151|      0|            } else {
  152|      0|                actual = self.parser.bytes_read();
  153|      0|            }
  154|       |
  155|       |            // we have not read a full frame yet, but we have read a full STOMP message, frame len is too long.
  156|      0|            if self.frame_lengths.len() == 0 {
  157|      0|                warn!("ws.frame_len too long {} > {}", self.ws.frame_len(), actual);
  158|      0|                if log_enabled!(Debug) {
  159|      0|                    debug!(
  160|      0|                        "header={}, text-body={}, bin-body={}",
  161|      0|                        self.parser.bytes_read(),
  162|      0|                        self.text_body_parser.bytes_read(),
  163|      0|                        self.fixed_length_body_parser.expected_len()
  164|       |                    );
  165|      0|                    debug!("msg_hdrs={}", self.parser.message.hdrs_as_string());
  166|      0|                }
  167|       |                // cant recover from this
  168|      0|                return false;
  169|      0|            }
  170|      0|
  171|      0|            let expected = self.frame_lengths.remove(0);
  172|      0|
  173|      0|            if expected != actual {
  174|      0|                warn!("Invalid ws.frame_len {} != {}", expected, actual);
  175|      0|                if log_enabled!(Debug) {
  176|      0|                    debug!(
  177|      0|                        "header={}, text-body={}, bin-body={}",
  178|      0|                        self.parser.bytes_read(),
  179|      0|                        self.text_body_parser.bytes_read(),
  180|      0|                        self.fixed_length_body_parser.expected_len()
  181|       |                    );
  182|      0|                    debug!("msg_hdrs={}", self.parser.message.hdrs_as_string());
  183|      0|                }
  184|       |                // is ws reader is happy and parser is happy we continue, must be a bug in length calculation
  185|      0|                return true;
  186|      0|            }
  187|      0|        }
  188|      0|        true
  189|      0|    }
  190|       |
  191|       |    /// send fully uploaded message to router for workflow
  192|       |    /// resets parser as a side effect
  193|      0|    fn route_message(&mut self) {
  194|      0|        let message = self.parser.take_message(Ownership::Session);
  195|      0|        //debug!("StompSession::route_message() {:?}", message);
  196|      0|
  197|      0|        let session = self.session.clone();
  198|      0|        router::route(message, session, self.web_socket);
  199|      0|    }
  200|       |
  201|       |    /// setup the correct body parser depending on hdrs
  202|      0|    fn setup_body_parser(&mut self) {
  203|      0|        let message = &self.parser.message;
  204|      0|        match message.get_header("content-length") {
  205|      0|            Some(content_length) => {
  206|      0|                match content_length.parse::<usize>() {
  207|      0|                    Ok(content_length) => {
  208|      0|                        if content_length > CONFIG.max_message_size {
  209|      0|                            // TODO could drain here, so its not fatal
  210|      0|                            self.session
  211|      0|                                .write()
  212|      0|                                .unwrap()
  213|      0|                                .send_client_error_fatal(ClientError::BodyFlup);
  214|      0|                        }
  215|      0|                        self.mode = ReaderMode::ReadingBinaryBody;
  216|      0|                        self.text_body_parser.reinit(CONFIG.max_message_size);
  217|      0|                        self.fixed_length_body_parser.reinit(content_length);
  218|       |                    }
  219|      0|                    Err(_e) => {
  220|      0|                        // protocol error here, send syntax error and bomb the tcp connection
  221|      0|                        self.session
  222|      0|                            .write()
  223|      0|                            .unwrap()
  224|      0|                            .send_client_error_fatal(ClientError::Syntax);
  225|      0|                    }
  226|       |                }
  227|       |            }
  228|       |            // no content-length means body must be \0 terminated
  229|      0|            None => {
  230|      0|                self.mode = ReaderMode::ReadingTextBody;
  231|      0|                self.text_body_parser.reinit(CONFIG.max_message_size);
  232|      0|                self.fixed_length_body_parser.reinit(0);
  233|      0|            }
  234|       |        }
  235|      0|    }
  236|       |
  237|       |    /// After reading a Command or getting to the end of a body there can be data left in buf
  238|       |    /// that is as yet unparsed. This method shifts the remaining data to the front of the buffer.
  239|       |    /// ensures each message has space for all the headers.
  240|      0|    fn realign_buffer(&mut self) {
  241|      0|        if self.end < self.pos {
  242|      0|            println!("WTF?? {} {}", self.pos, self.end);
  243|      0|        }
  244|      0|        let len = self.end - self.pos;
  245|      0|        self.buf.copy_within(self.pos..self.end, 0);
  246|      0|        self.pos = 0;
  247|      0|        self.end = len;
  248|      0|    }
  249|       |
  250|       |    /// Called either when we have been pushed extra data from the network, or when
  251|       |    /// we processed only part of the data and there is some left between self.pos and self.end
  252|      0|    fn process_buffer_portion(&mut self) -> Result<(), ParserState> {
  253|      0|        if self.pos == self.end {
  254|       |            // read something but not STOMP data
  255|      0|            return Ok(());
  256|      0|        }
  257|      0|
  258|      0|        let chunk = &self.buf[self.pos..self.end];
  259|      0|
  260|      0|        match self.mode {
  261|      0|            ReaderMode::ReadingTextBody => {
  262|      0|                match self.text_body_parser.push(chunk, &mut self.parser.message) {
  263|      0|                    Ok(read) => {
  264|      0|                        if self.text_body_parser.is_done() {
  265|      0|                            self.pos = read;
  266|      0|                            return self.handle_message_received();
  267|       |                        } else {
  268|      0|                            self.pos = 0;
  269|      0|                            self.end = 0;
  270|      0|                            return Ok(());
  271|       |                        }
  272|       |                    }
  273|      0|                    Err(ps) => {
  274|      0|                        debug!("body parser error {:?}", ps);
  275|      0|                        if ps == BodyFlup {
  276|      0|                            return Err(ParserState::BodyFlup);
  277|      0|                        }
  278|      0|                        return Err(ParserState::InvalidMessage);
  279|       |                    }
  280|       |                }
  281|       |            }
  282|       |
  283|       |            ReaderMode::ReadingBinaryBody => {
  284|      0|                match self
  285|      0|                    .fixed_length_body_parser
  286|      0|                    .push(chunk, &mut self.parser.message)
  287|       |                {
  288|      0|                    Ok(read) => {
  289|      0|                        if self.fixed_length_body_parser.is_done() {
  290|      0|                            self.pos = read;
  291|      0|                            return self.handle_message_received();
  292|       |                        } else {
  293|      0|                            self.pos = 0;
  294|      0|                            self.end = 0;
  295|      0|                            return Ok(());
  296|       |                        }
  297|       |                    }
  298|      0|                    Err(ps) => {
  299|      0|                        debug!("body parser error {:?}", ps);
  300|      0|                        if ps == BodyFlup {
  301|      0|                            return Err(ParserState::BodyFlup);
  302|      0|                        }
  303|      0|                        return Err(ParserState::InvalidMessage);
  304|       |                    }
  305|       |                }
  306|       |            }
  307|       |
  308|       |            ReaderMode::ReadingCommand => {
  309|      0|                match self.parser.push(&self.buf, self.pos, chunk) {
  310|      0|                    Ok(ParserState::Again) => {
  311|      0|                        self.pos += chunk.len();
  312|      0|                        if self.pos != self.end {
  313|      0|                            debug!("pos not what it ought ta be");
  314|      0|                        }
  315|      0|                        return Ok(());
  316|       |                    }
  317|      0|                    Ok(ParserState::Done(read)) => {
  318|      0|                        self.pos = read;
  319|      0|                        return self.handle_command_received();
  320|       |                    }
  321|      0|                    Ok(ParserState::Message(read)) => {
  322|      0|                        // loop
  323|      0|                        self.pos = read;
  324|      0|                        return self.handle_command_received();
  325|       |                    }
  326|      0|                    Ok(ps) => {
  327|      0|                        panic!("unreachable {:?}", ps)
  328|       |                    }
  329|      0|                    Err(ps) => {
  330|      0|                        return Err(ps);
  331|       |                    }
  332|       |                }
  333|       |            }
  334|       |        }
  335|      0|    }
  336|       |
  337|       |    /// Called prior to returning Err() which is our last breath
  338|      0|    fn terminated(&self) {
  339|      0|        self.session.write().unwrap().read_terminated();
  340|      0|    }
  341|       |}
  342|       |
  343|       |impl Future for Reader {
  344|       |    type Item = ();
  345|       |    type Error = ();
  346|       |
  347|      0|    fn poll(&mut self) -> Result<Async<Self::Item>, Self::Error> {
  348|      0|        debug!("read polled"); // called on connect, data, and on zero bytes
  349|       |
  350|      0|        if self.session.read().unwrap().shutdown_pending() {
  351|      0|            debug!("reader closed");
  352|      0|            self.terminated();
  353|      0|            return Err(());
  354|      0|        }
  355|       |
  356|      0|        loop {
  357|      0|            if !self.read_done {
  358|       |                // &mut because poll_read reads into the buffer,
  359|       |                // this only writes to pos
  360|      0|                let remain = self.buf.len() - self.pos;
  361|      0|
  362|      0|                match self.read_half.poll_read(&mut self.buf[self.pos..remain]) {
  363|      0|                    Ok(Async::Ready(n)) => {
  364|      0|                        if n == 0 {
  365|       |                            // TcpConnection closed
  366|      0|                            self.read_done = true;
  367|      0|                            debug!("tcp connection closed");
  368|      0|                            self.session.write().unwrap().shutdown();
  369|       |                        } else {
  370|      0|                            if self.pos != self.end {
  371|      0|                                debug!("pos not what it ought ta be");
  372|      0|                                self.pos = self.end;
  373|      0|                            }
  374|       |
  375|      0|                            self.end += n;
  376|      0|
  377|      0|                            // WebSocket un-munging
  378|      0|                            if self.web_socket
  379|      0|                                || self.session.read().unwrap().get_flag(FLAG_WEB_SOCKETS)
  380|       |                            {
  381|      0|                                if !self.web_socket {
  382|      0|                                    self.web_socket = true;
  383|      0|                                }
  384|      0|                                match ws_demunge(
  385|      0|                                    &mut self.ws,
  386|      0|                                    &mut self.buf,
  387|      0|                                    self.pos,
  388|      0|                                    self.end,
  389|      0|                                    &mut self.frame_lengths,
  390|      0|                                ) {
  391|      0|                                    Ok((new_pos, new_end)) => {
  392|      0|                                        self.pos = new_pos;
  393|      0|                                        self.end = new_end;
  394|      0|                                        // potentially here we did not process any STOMP data
  395|      0|                                    }
  396|       |                                    Err(_) => {
  397|      0|                                        self.session
  398|      0|                                            .write()
  399|      0|                                            .unwrap()
  400|      0|                                            .read_error(ParserState::InvalidMessage);
  401|      0|                                        self.terminated();
  402|      0|                                        return Err(());
  403|       |                                    }
  404|       |                                }
  405|      0|                            }
  406|       |
  407|      0|                            match self.process_buffer_portion() {
  408|      0|                                Err(ps) => {
  409|      0|                                    debug!("reader closed {:?}", ps);
  410|      0|                                    self.session.write().unwrap().read_error(ps);
  411|      0|                                    self.terminated();
  412|      0|                                    return Err(());
  413|       |                                }
  414|      0|                                Ok(()) => {
  415|      0|                                    self.session.read().unwrap().read_something();
  416|      0|                                    // loop
  417|      0|                                }
  418|       |                            }
  419|       |                        }
  420|       |                    }
  421|       |                    Ok(Async::NotReady) => {
  422|      0|                        return Ok(Async::NotReady);
  423|       |                    }
  424|      0|                    Err(_e) => {
  425|      0|                        debug!("reader closed");
  426|      0|                        self.terminated();
  427|      0|                        return Err(());
  428|       |                    }
  429|       |                };
  430|      0|            }
  431|       |
  432|      0|            if self.read_done {
  433|      0|                debug!("reader closed");
  434|      0|                self.terminated();
  435|      0|                return Err(());
  436|      0|            }
  437|       |        }
  438|      0|    }
  439|       |}
  440|       |
  441|       |// elaborate way to be able to notify read thread when we want to tell it to die
  442|       |
  443|       |pub struct ReadKiller {
  444|       |    pub die: bool,
  445|       |    task: Option<Task>,
  446|       |}
  447|       |
  448|       |impl ReadKiller {
  449|      0|    pub fn new() -> ReadKiller {
  450|      0|        ReadKiller {
  451|      0|            die: false,
  452|      0|            task: None,
  453|      0|        }
  454|      0|    }
  455|       |
  456|      0|    pub fn kill(&mut self) {
  457|      0|        self.die = true;
  458|      0|        match &self.task {
  459|      0|            Some(task) => {
  460|      0|                debug!("notifying read to stop it");
  461|       |                // seems this might not do anything if we get here from a read
  462|      0|                task.notify();
  463|      0|                self.task = None;
  464|       |            }
  465|      0|            None => {}
  466|       |        }
  467|      0|    }
  468|       |}
  469|       |
  470|       |impl Future for ReadKiller {
  471|       |    type Item = ();
  472|       |    type Error = ();
  473|      0|    fn poll(&mut self) -> Result<Async<Self::Item>, Self::Error> {
  474|      0|        debug!("read killer polled");
  475|      0|        self.task = Some(futures::task::current());
  476|      0|        match self.die {
  477|       |            true => {
  478|      0|                debug!("read killer closed");
  479|      0|                self.task = None;
  480|      0|                Err(())
  481|       |            }
  482|      0|            _ => Ok(Async::NotReady),
  483|       |        }
  484|      0|    }
  485|       |}

src/session/stomp_session.rs:
    1|       |use std::sync::atomic::{AtomicI64, Ordering};
    2|       |use std::sync::{Arc, RwLock};
    3|       |
    4|       |use log::*;
    5|       |
    6|       |use chashmap::CHashMap;
    7|       |use chrono::Utc;
    8|       |
    9|       |use tokio::io::WriteHalf;
   10|       |use tokio::net::TcpStream;
   11|       |use tokio::prelude::task::Task;
   12|       |use tokio::prelude::*;
   13|       |
   14|       |use crate::downstream::DownstreamConnector;
   15|       |use crate::errors::*;
   16|       |use crate::init::CONFIG;
   17|       |use crate::init::SERVER;
   18|       |use crate::message::response::*;
   19|       |use crate::message::stomp_message::{Header, StompMessage};
   20|       |use crate::parser::ParserState;
   21|       |use crate::session::mq::{Mq, SessionMessage};
   22|       |use crate::session::reader::{ReadKiller, Reader};
   23|       |use crate::session::subscription::Subscription;
   24|       |use crate::session::writer::Writer;
   25|       |use crate::web_socket::ws_ports::{ws_is_trusted_port, ws_is_web_port};
   26|       |use std::ops::Deref;
   27|       |
   28|       |// network may take some time to deliver a keep-alive
   29|       |const READ_TIMEOUT_MARGIN: i64 = 15000;
   30|       |
   31|       |/// Session upgraded an HTTP connection and is using websockets
   32|       |pub const FLAG_WEB_SOCKETS: u64 = 1;
   33|       |/// Session started on wep ports
   34|       |pub const FLAG_WEB: u64 = 2;
   35|       |/// user has admin privs
   36|       |pub const FLAG_ADMIN: u64 = 4;
   37|       |/// session is downstream of xtomp
   38|       |pub const FLAG_DOWNSTREAM: u64 = 8;
   39|       |
   40|       |///
   41|       |/// Stomp session is the god object for a single users/connection
   42|       |///
   43|       |/// read and write operations keep a handle on the Session.  Everything it holds must be Send safe
   44|       |///
   45|       |
   46|       |pub struct StompSession {
   47|       |    id: usize,
   48|       |    /// existence of this value implies authentication
   49|       |    pub user: Option<String>,
   50|       |    /// bit mask of flags
   51|       |    flags: u64,
   52|       |    // benefit of RwLock is we don't need a write or mut on self to push messages to mq
   53|       |    mq: Arc<RwLock<Mq>>,
   54|       |    write_half: Option<Arc<RwLock<WriteHalf<TcpStream>>>>,
   55|       |    read_killer: Option<Arc<RwLock<ReadKiller>>>,
   56|       |    // TODO Vec<DownstreamConnector> but I cannot find the rust syntax for thread safe owned traits
   57|       |    pub(crate) downstream_connector: Option<Arc<RwLock<DownstreamConnector>>>,
   58|       |
   59|       |    timeout_task: Option<Task>,
   60|       |    pub heart_beat_read: u32,
   61|       |    pub heart_beat_write: u32,
   62|       |    last_read: AtomicI64,
   63|       |    last_write: AtomicI64,
   64|       |
   65|       |    /// have been asked for graceful shutdown
   66|       |    shutdown: bool,
   67|       |    subscriptions: Vec<Arc<RwLock<Subscription>>>,
   68|       |    pending_acks: CHashMap<usize, usize>,
   69|       |}
   70|       |
   71|       |/// Session is 1:1 with a TCP connection in STOMP
   72|       |impl StompSession {
   73|      8|    pub fn new() -> StompSession {
   74|      8|        let mut session = StompSession {
   75|      8|            id: SERVER.new_session(),
   76|      8|            user: None,
   77|      8|            flags: 0,
   78|      8|            mq: Arc::new(RwLock::new(Mq::new())),
   79|      8|            write_half: None,
   80|      8|            read_killer: None,
   81|      8|            downstream_connector: None,
   82|      8|            timeout_task: None,
   83|      8|            heart_beat_read: 0,
   84|      8|            heart_beat_write: 0,
   85|      8|            last_read: AtomicI64::new(Utc::now().timestamp_millis()),
   86|      8|            last_write: AtomicI64::new(Utc::now().timestamp_millis()),
   87|      8|            shutdown: false,
   88|      8|            subscriptions: vec![],
   89|      8|            pending_acks: CHashMap::new(),
   90|      8|        };
   91|      8|        session.set_heart_beat_defaults();
   92|      8|        session
   93|      8|    }
   94|       |
   95|       |    /// Unique session ID. Remains unique for the lifetime of the process.
   96|      0|    pub fn id(&self) -> usize {
   97|      0|        self.id
   98|      0|    }
   99|       |
  100|       |    /// returns login/user name or ""
  101|       |    pub fn user(&self) -> &str {
  102|      8|        if let Some(user) = &self.user {
  103|      0|            return user.as_str();
  104|      8|        }
  105|      8|        ""
  106|      8|    }
  107|       |
  108|       |    /// Returns a flag from a u64 bitmask of flags.  Currently this is the only data that may be associated
  109|       |    /// with a session.
  110|      3|    pub fn get_flag(&self, flag_mask: u64) -> bool {
  111|      3|        self.flags & flag_mask == flag_mask
  112|      3|    }
  113|       |
  114|       |    /// Sets a flag from a u64 bitmask of flags
  115|      1|    pub fn set_flag(&mut self, flag_mask: u64) {
  116|      1|        self.flags |= flag_mask
  117|      1|    }
  118|       |
  119|      0|    pub(crate) fn split(
  120|      0|        &mut self,
  121|      0|        sock: TcpStream,
  122|      0|        session: Arc<RwLock<StompSession>>,
  123|      0|    ) -> (Reader, Writer) {
  124|      0|        {
  125|      0|            if !self.get_flag(FLAG_DOWNSTREAM) {
  126|      0|                let port = sock.local_addr().unwrap().port();
  127|      0|                if ws_is_trusted_port(port) {
  128|      0|                    self.set_flag(FLAG_ADMIN);
  129|      0|                }
  130|      0|                if ws_is_web_port(port) {
  131|      0|                    self.set_flag(FLAG_WEB);
  132|      0|                }
  133|      0|            }
  134|       |        }
  135|      0|        let (read_half, write_half) = sock.split();
  136|      0|
  137|      0|        let write_half_lock = Arc::new(RwLock::new(write_half));
  138|      0|
  139|      0|        self.set_write_half(write_half_lock.clone());
  140|      0|
  141|      0|        (
  142|      0|            Reader::new(session.clone(), self.id(), read_half),
  143|      0|            Writer::new(session.clone(), self.id(), write_half_lock),
  144|      0|        )
  145|      0|    }
  146|       |
  147|       |    // I/O
  148|       |
  149|      0|    fn set_write_half(&mut self, write_half: Arc<RwLock<WriteHalf<TcpStream>>>) {
  150|      0|        self.write_half = Some(write_half);
  151|      0|    }
  152|       |
  153|      0|    pub(crate) fn set_read_killer(&mut self, read_killer: Arc<RwLock<ReadKiller>>) {
  154|      0|        self.read_killer = Some(read_killer);
  155|      0|    }
  156|       |
  157|       |    // Mq
  158|       |
  159|       |    /// poll Mq until it has a message to write, or it has been drained or closed
  160|      0|    pub(crate) fn poll_mq(&self) -> Result<Async<()>, ()> {
  161|      0|        match self.mq.read().unwrap().poll() {
  162|      0|            Ok(Async::Ready(())) => Ok(Async::Ready(())),
  163|      0|            Ok(Async::NotReady) => Ok(Async::NotReady),
  164|      0|            Err(_) => Err(()),
  165|       |        }
  166|      0|    }
  167|       |
  168|      0|    pub(crate) fn set_mq_task(&self, task: Task) {
  169|      0|        self.mq.write().unwrap().set_task(task);
  170|      0|    }
  171|       |
  172|       |    /// dont call this unless poll returned Ok(Ready)
  173|      0|    pub(crate) fn pop(&self) -> Option<SessionMessage> {
  174|      0|        self.mq.write().unwrap().next()
  175|      0|    }
  176|       |
  177|       |    /// returns mq length, i.e. number of pending messages
  178|      0|    pub fn len(&self) -> usize {
  179|      0|        self.mq.read().unwrap().len()
  180|      0|    }
  181|       |
  182|       |    // Mq messaging
  183|       |
  184|       |    /// Push an error message to a session, fails, and returns false, if a write lock can not be obtained.
  185|       |    pub fn send_client_error(&self, err: ClientError) -> bool {
  186|      0|        return if let Ok(mut mq) = self.mq.try_write() {
  187|      0|            mq.push(get_response_error_client(err), vec![]);
  188|      0|            true
  189|       |        } else {
  190|      0|            false
  191|       |        };
  192|      0|    }
  193|       |
  194|       |    /// Push an error message to a session, fails, and returns false, if a write lock can not be obtained.
  195|       |    /// Then calls `shutdown()` to indicate that existing messages should be delivered and the TCP connection closed.
  196|       |    pub fn send_client_error_fatal(&mut self, err: ClientError) -> bool {
  197|       |        let sent;
  198|      0|        if let Ok(mut mq) = self.mq.try_write() {
  199|      0|            mq.push(get_response_error_client(err), vec![]);
  200|      0|            sent = true
  201|      0|        } else {
  202|      0|            sent = false;
  203|      0|        }
  204|      0|        self.shutdown();
  205|      0|        sent
  206|      0|    }
  207|       |
  208|       |    /// Send a message to this session only.
  209|       |    pub fn send_message(&self, message: Arc<StompMessage>) -> bool {
  210|      0|        return if let Ok(mut mq) = self.mq.try_write() {
  211|      0|            mq.push(message, vec![]);
  212|      0|            true
  213|       |        } else {
  214|      0|            false
  215|       |        };
  216|      0|    }
  217|       |
  218|       |    /// Send a message with additional headers.
  219|       |    pub fn send_message_w_hdrs(&self, message: Arc<StompMessage>, headers: Vec<Header>) -> bool {
  220|      0|        return if let Ok(mut mq) = self.mq.try_write() {
  221|      0|            mq.push(message, headers);
  222|      0|            true
  223|       |        } else {
  224|      0|            false
  225|       |        };
  226|      0|    }
  227|       |
  228|       |    // Subscriptions
  229|       |
  230|       |    /// returns the number of subscriptions to destinations this session has.
  231|      0|    pub fn count_subscription(&self) -> usize {
  232|      0|        self.subscriptions.len()
  233|      0|    }
  234|       |
  235|      0|    pub(crate) fn add_subscription(&mut self, sub: Arc<RwLock<Subscription>>) {
  236|      0|        self.subscriptions.push(sub);
  237|      0|        debug!("subscription count {}", self.count_subscription());
  238|      0|    }
  239|       |
  240|      0|    pub(crate) fn remove_subscription(&mut self, id: u64) {
  241|      0|        self.subscriptions
  242|      0|            .retain(|sub| sub.read().unwrap().subscription_id() != id);
  243|      0|    }
  244|       |
  245|      0|    pub fn unsubscribe(&mut self, id: u64) {
  246|      0|        for sub in &self.subscriptions {
  247|      0|            let sub = sub.read().unwrap();
  248|      0|            if sub.subscription_id() == id {
  249|      0|                sub.unsubscribe();
  250|      0|                break;
  251|      0|            }
  252|       |        }
  253|      0|        self.remove_subscription(id);
  254|      0|    }
  255|       |
  256|      0|    pub fn unsubscribe_all(&mut self) {
  257|      0|        for sub in &self.subscriptions {
  258|      0|            {
  259|      0|                let sub = sub.read().unwrap();
  260|      0|                sub.unsubscribe();
  261|      0|            }
  262|       |        }
  263|       |        // WTF this causes deadlock!!
  264|      0|        self.subscriptions.clear();
  265|      0|    }
  266|       |
  267|       |    // ACK that the client should send us. Annoyingly, STOMP spec does not expect
  268|       |    // the client to remind us which destination a message came from so we have to keep state.
  269|      0|    pub(crate) fn pending_ack(&self, msg_id: usize, destination_id: usize) {
  270|      0|        self.pending_acks.insert(msg_id, destination_id);
  271|      0|    }
  272|       |
  273|       |    fn get_destination_id(&self, msg_id: usize) -> Option<usize> {
  274|      0|        if let Some(entry) = self.pending_acks.get(&msg_id) {
  275|      0|            return Some(*entry.deref());
  276|      0|        }
  277|      0|        return None;
  278|      0|    }
  279|       |
  280|       |    pub fn ack(&self, ack_nack: bool, msg_id: &String) -> bool {
  281|      0|        if let Ok(msg_id) = msg_id.parse::<usize>() {
  282|      0|            if let Some(destination_id) = self.get_destination_id(msg_id) {
  283|      0|                if let Some(destination) = SERVER.find_destination_by_id(&destination_id) {
  284|      0|                    let destination = destination.read().unwrap();
  285|      0|                    if !destination.auto_ack() {
  286|      0|                        debug!("acking msg @ dest={} msg_id={}", destination.name(), msg_id);
  287|      0|                        if ack_nack {
  288|      0|                            destination.ack(msg_id);
  289|      0|                            return true;
  290|       |                        } else {
  291|      0|                            destination.nack(msg_id);
  292|      0|                            return true;
  293|       |                        }
  294|      0|                    }
  295|      0|                }
  296|      0|            }
  297|      0|        }
  298|      0|        return false;
  299|      0|    }
  300|       |
  301|       |    // Session State
  302|       |
  303|      8|    pub fn set_heart_beat_defaults(&mut self) {
  304|      8|        self.heart_beat_read = CONFIG.heart_beat_read;
  305|      8|        self.heart_beat_write = CONFIG.heart_beat_write_min
  306|      8|            + ((CONFIG.heart_beat_write_max - CONFIG.heart_beat_write_min) / 2);
  307|      8|    }
  308|       |
  309|       |    /// This effectively authenticates the session.
  310|      0|    pub fn set_login(&mut self, name: &str) -> bool {
  311|      0|        self.user = Some(String::from(name));
  312|      0|        info!("login usr={}", self.user());
  313|      0|        true
  314|      0|    }
  315|       |
  316|       |    /// Returns true if we have a login name.
  317|      0|    pub fn is_authenticated(&self) -> bool {
  318|      0|        match self.user {
  319|      0|            Some(_) => true,
  320|      0|            _ => false,
  321|       |        }
  322|      0|    }
  323|       |
  324|       |    /// Upgrade the TCP connection from HTTP to STOMP over websockets.
  325|      0|    pub(crate) fn ws_upgrade(&mut self) {
  326|      0|        self.set_flag(FLAG_WEB_SOCKETS);
  327|      0|        // converts to web permissions on upgrade
  328|      0|        self.set_flag(FLAG_WEB);
  329|      0|    }
  330|       |
  331|       |    // Lifecycle
  332|       |
  333|       |    /// Returns true if shutdown has been called.
  334|      0|    pub fn shutdown_pending(&self) -> bool {
  335|      0|        self.shutdown
  336|      0|    }
  337|       |
  338|       |    /// Initiate graceful shutdown, publishing messages left on mq first.
  339|      0|    pub fn shutdown(&mut self) {
  340|      0|        info!("session shutdown usr={} id={}", self.user(), self.id);
  341|      0|        self.shutdown = true;
  342|      0|
  343|      0|        self.unsubscribe_all();
  344|      0|
  345|      0|        match self.mq.write() {
  346|      0|            Ok(mut mq) => match mq.drain() {
  347|      0|                Ok(_) => {
  348|      0|                    debug!("empty mq at drain id={}", self.id);
  349|      0|                    if let Some(write_half) = &self.write_half {
  350|      0|                        if let Ok(mut write_half) = write_half.try_write() {
  351|      0|                            write_half.shutdown().ok();
  352|      0|                        }
  353|      0|                    }
  354|      0|                    self.write_half = None;
  355|       |                }
  356|      0|                Err(remaining) => {
  357|      0|                    debug!(
  358|      0|                        "session will shutdown when messages are drained: {} id={}",
  359|      0|                        remaining, self.id
  360|       |                    );
  361|       |                }
  362|       |            },
  363|      0|            Err(_) => warn!("mq.drain() lock failed"),
  364|       |        }
  365|       |
  366|      0|        if let Some(read_killer) = &self.read_killer {
  367|      0|            read_killer.write().unwrap().kill();
  368|      0|            // un hook read
  369|      0|            self.read_killer = None;
  370|      0|        } else {
  371|      0|            debug!("no read_killer?? '{}'", self.user());
  372|       |        }
  373|       |
  374|      0|        if let Some(timeout_task) = &self.timeout_task {
  375|      0|            timeout_task.notify();
  376|      0|        }
  377|      0|        debug!("session shutdown complete id={}", self.id);
  378|      0|    }
  379|       |
  380|       |    /// Initiate ungraceful shutdown.  Pending messages are discarded.
  381|      0|    pub fn kill(&mut self) {
  382|      0|        info!("session kill usr={}", self.user());
  383|      0|        self.shutdown = true;
  384|      0|
  385|      0|        self.unsubscribe_all();
  386|      0|
  387|      0|        match self.mq.write() {
  388|      0|            Ok(mut mq) => {
  389|      0|                mq.close();
  390|      0|                if let Some(write_half) = &self.write_half {
  391|      0|                    if let Ok(mut write_half) = write_half.try_write() {
  392|      0|                        write_half.shutdown().ok();
  393|      0|                    }
  394|      0|                }
  395|      0|                self.write_half = None;
  396|       |            }
  397|      0|            Err(_) => warn!("mq.close() lock failed"),
  398|       |        }
  399|       |
  400|      0|        if let Some(read_killer) = &self.read_killer {
  401|      0|            read_killer.write().unwrap().kill();
  402|      0|            self.read_killer = None;
  403|      0|        } else {
  404|      0|            debug!("no read_killer on kill");
  405|       |        }
  406|       |
  407|      0|        if let Some(timeout_task) = &self.timeout_task {
  408|      0|            timeout_task.notify();
  409|      0|        }
  410|      0|    }
  411|       |
  412|       |    /// called when the writer ends because mq is drained
  413|      0|    pub(crate) fn write_terminated(&mut self) {
  414|      0|        if let Some(write_half) = &self.write_half {
  415|      0|            if let Ok(mut write_half) = write_half.try_write() {
  416|      0|                write_half.shutdown().ok();
  417|      0|            }
  418|      0|        }
  419|      0|        self.write_half = None;
  420|      0|    }
  421|       |
  422|       |    /// called when the reader ends
  423|       |    pub(crate) fn read_terminated(&mut self) {
  424|      0|        if let Some(read_killer) = &self.read_killer {
  425|      0|            read_killer.write().unwrap().kill();
  426|      0|            // un hook read
  427|      0|            self.read_killer = None;
  428|       |            // session termination determined as reading ended.
  429|      0|            if let Some(shutdown_listener) = &self.downstream_connector {
  430|      0|                shutdown_listener.read().unwrap().session_shutdown(
  431|      0|                    shutdown_listener.clone(),
  432|      0|                    self.id,
  433|      0|                    self.flags,
  434|      0|                );
  435|      0|            }
  436|      0|            self.downstream_connector = None;
  437|      0|        }
  438|      0|    }
  439|       |
  440|       |    /// called when we had problems reading the input stream
  441|      0|    pub(crate) fn read_error(&mut self, ps: ParserState) {
  442|      0|        if let Ok(mut mq) = self.mq.try_write() {
  443|      0|            if ps == ParserState::BodyFlup {
  444|      0|                mq.push(get_response_error_client(ClientError::BodyFlup), vec![]);
  445|      0|            } else if ps == ParserState::HdrFlup {
  446|      0|                mq.push(get_response_error_client(ClientError::HdrFlup), vec![]);
  447|      0|            } else {
  448|      0|                mq.push(get_response_error_client(ClientError::Syntax), vec![]);
  449|      0|            }
  450|      0|        }
  451|      0|        self.shutdown();
  452|      0|    }
  453|       |
  454|       |    /// called when we had problems writing
  455|      0|    pub(crate) fn write_error(&mut self) {
  456|      0|        self.kill();
  457|      0|    }
  458|       |
  459|       |    // Timeout handling
  460|       |
  461|       |    /// fired for either read or write timeout
  462|       |    /// read timeout results in shutdown()
  463|       |    /// write timeout results in a heart-beat
  464|      0|    pub(crate) fn timeout(&mut self) -> Result<(), tokio::timer::Error> {
  465|      0|        debug!("timeout polled");
  466|      0|        if let None = self.timeout_task {
  467|      0|            self.timeout_task = Some(futures::task::current());
  468|      0|        }
  469|      0|        if self.shutdown {
  470|      0|            debug!("timeout ending");
  471|      0|            self.timeout_task = None;
  472|      0|            return Err(tokio::timer::Error::shutdown());
  473|      0|        }
  474|      0|
  475|      0|        // read timeout
  476|      0|        let last_read = self.last_read.load(Ordering::Relaxed);
  477|      0|        if last_read + (self.heart_beat_read as i64) + READ_TIMEOUT_MARGIN
  478|      0|            < Utc::now().timestamp_millis()
  479|       |        {
  480|      0|            warn!("read timeout at {}", self.heart_beat_read);
  481|      0|            self.shutdown();
  482|      0|            return Ok(());
  483|      0|        }
  484|      0|
  485|      0|        // write timeout
  486|      0|        if self.should_heart_beat() {
  487|      0|            if let Ok(mq) = self.mq.try_read() {
  488|      0|                mq.notify();
  489|      0|            }
  490|      0|        }
  491|       |
  492|      0|        return Ok(());
  493|      0|    }
  494|       |
  495|       |    /// return true if its time to send a \n heart beat
  496|      5|    pub(crate) fn should_heart_beat(&self) -> bool {
  497|      5|        let last_write = self.last_write.load(Ordering::Relaxed);
  498|      5|        return last_write + (self.heart_beat_write as i64) < Utc::now().timestamp_millis();
  499|      5|    }
  500|       |
  501|       |    /// return number of milliseconds until the next desired timeout
  502|      2|    pub fn next_timeout(&self) -> i64 {
  503|      2|        let next_write = self.last_write.load(Ordering::Relaxed) + (self.heart_beat_write as i64);
  504|      2|        let next_read = self.last_read.load(Ordering::Relaxed)
  505|      2|            + (self.heart_beat_read as i64)
  506|      2|            + READ_TIMEOUT_MARGIN;
  507|      2|        return match next_read < next_write {
  508|      0|            true => next_read - Utc::now().timestamp_millis(),
  509|      2|            false => next_write - Utc::now().timestamp_millis(),
  510|       |        };
  511|      2|    }
  512|       |
  513|       |    // N.B. not mut
  514|       |
  515|      0|    pub(crate) fn read_something(&self) {
  516|      0|        self.last_read
  517|      0|            .store(Utc::now().timestamp_millis(), Ordering::Relaxed);
  518|      0|    }
  519|       |
  520|      2|    pub(crate) fn wrote_something(&self) {
  521|      2|        self.last_write
  522|      2|            .store(Utc::now().timestamp_millis(), Ordering::Relaxed);
  523|      2|    }
  524|       |}
  525|       |
  526|       |impl Drop for StompSession {
  527|      8|    fn drop(&mut self) {
  528|      8|        debug!("dropped session for '{}' id={}", self.user(), self.id);
  529|      8|        SERVER.drop_session();
  530|      8|    }
  531|       |}
  532|       |
  533|       |pub trait ShutdownListener {
  534|       |    fn session_shutdown(&self, id: usize, flags: u64);
  535|       |}
  536|       |
  537|       |#[cfg(test)]
  538|       |mod tests {
  539|       |    use super::*;
  540|       |    use std::{thread, time};
  541|       |
  542|      1|    #[test]
  543|      1|    fn test_new() {
  544|      1|        let mut session = StompSession::new();
  545|      1|        assert_eq!(120000, session.heart_beat_read);
  546|      1|        assert_eq!(120000, session.heart_beat_write);
  547|      1|        session.set_flag(FLAG_ADMIN);
  548|      1|        assert_eq!(true, session.get_flag(FLAG_ADMIN));
  549|      1|        assert_eq!(false, session.get_flag(FLAG_WEB));
  550|      1|        assert_eq!(false, session.get_flag(FLAG_WEB_SOCKETS));
  551|       |
  552|       |        // heart-beat tests
  553|      1|        assert_eq!(false, session.should_heart_beat());
  554|       |        // set silly 10ms timeout so this test runs quickly
  555|      1|        session.heart_beat_write = 10;
  556|      1|        thread::sleep(time::Duration::from_millis(11));
  557|      1|        assert_eq!(true, session.should_heart_beat());
  558|      1|        session.wrote_something();
  559|      1|        assert_eq!(false, session.should_heart_beat());
  560|       |
  561|      1|        thread::sleep(time::Duration::from_millis(11));
  562|      1|        assert_eq!(true, session.should_heart_beat());
  563|      1|        session.wrote_something();
  564|      1|        assert_eq!(false, session.should_heart_beat());
  565|       |
  566|      1|        println!("next_timeout = {}", session.next_timeout());
  567|      1|        thread::sleep(time::Duration::from_millis(5));
  568|      1|        println!("next_timeout = {}", session.next_timeout());
  569|      1|    }
  570|       |}

src/session/subscription.rs:
    1|       |use std::sync::{Arc, RwLock};
    2|       |
    3|       |use log::*;
    4|       |
    5|       |use chrono::{DateTime, Utc};
    6|       |
    7|       |use crate::message::stomp_message::{Header, StompMessage};
    8|       |use crate::session::filter::Filter;
    9|       |use crate::session::stomp_session::StompSession;
   10|       |use crate::util::rollover_gt;
   11|       |use crate::workflow::destination::destination::Destination;
   12|       |use std::ops::Deref;
   13|       |
   14|      0|#[derive(Debug, PartialEq)]
   15|       |pub enum Ack {
   16|       |    Auto,
   17|       |    Client,
   18|       |    ClientIndividual,
   19|       |}
   20|       |
   21|      0|pub fn parse_ack_hdr(hdr: &String) -> Ack {
   22|      0|    match hdr.as_str() {
   23|      0|        "client" => return Ack::Client,
   24|      0|        "client-individual" => return Ack::ClientIndividual,
   25|      0|        _ => Ack::Auto,
   26|       |    }
   27|      0|}
   28|       |
   29|       |/// Subscription to a destination, these are stored on the Destination & the session
   30|       |/// and hold references to the session, so messages can be queued
   31|       |// TODO could be weak references so we don't have to manually unhook on shutdown
   32|       |
   33|       |pub struct Subscription {
   34|       |    session: Arc<RwLock<StompSession>>,
   35|       |    destination: Arc<RwLock<Destination>>,
   36|       |    filter: Option<Filter>,
   37|       |    filter_self: bool,
   38|       |    id: u64, // subscription id
   39|       |    hash_id: String,
   40|       |    last_msg: usize,          // last delivered message
   41|       |    timestamp: DateTime<Utc>, // start time
   42|       |    ack: Ack,                 // auto|client|client-individual
   43|       |}
   44|       |
   45|       |impl Subscription {
   46|      0|    pub fn new(
   47|      0|        session: Arc<RwLock<StompSession>>,
   48|      0|        destination: Arc<RwLock<Destination>>,
   49|      0|        id: u64,
   50|      0|        ack: Ack,
   51|      0|        filter: Option<Filter>,
   52|      0|        filter_self: bool,
   53|      0|    ) -> Subscription {
   54|      0|        let session_id = session.read().unwrap().id();
   55|      0|        debug!(
   56|      0|            "new subscription id={}, ack_mode={:?}, filter_self={}",
   57|      0|            id, ack, filter_self
   58|       |        );
   59|      0|        Subscription {
   60|      0|            session,
   61|      0|            destination,
   62|      0|            filter,
   63|      0|            filter_self,
   64|      0|            id,
   65|      0|            hash_id: format!("{}-{}", session_id, id),
   66|      0|            last_msg: 0,
   67|      0|            timestamp: Utc::now(),
   68|      0|            ack,
   69|      0|        }
   70|      0|    }
   71|       |
   72|      0|    pub fn hash_id(&self) -> &String {
   73|      0|        &self.hash_id
   74|      0|    }
   75|       |
   76|       |    pub fn user_id(&self) -> String {
   77|      0|        if let Some(user) = &self.session.read().unwrap().user {
   78|      0|            return user.clone();
   79|       |        } else {
   80|      0|            return String::from("");
   81|       |        }
   82|      0|    }
   83|       |
   84|      0|    pub fn subscription_id(&self) -> u64 {
   85|      0|        self.id
   86|      0|    }
   87|       |
   88|      0|    pub(crate) fn unsubscribe(&self) -> bool {
   89|      0|        self.destination.write().unwrap().unsubscribe(&self.hash_id)
   90|      0|    }
   91|       |
   92|       |    /// send a published message to the client, checks first to see if it was already published
   93|      0|    pub(crate) fn publish(
   94|      0|        &mut self,
   95|      0|        dest_id: usize,
   96|      0|        message: Arc<StompMessage>,
   97|      0|        do_filter: bool,
   98|      0|    ) -> bool {
   99|      0|        let msg_id;
  100|      0|        {
  101|      0|            msg_id = message.id;
  102|      0|            if do_filter {
  103|      0|                if let Some(filter) = &self.filter {
  104|      0|                    if !filter.matches_message(message.deref()) {
  105|      0|                        debug!("filtered out {}", msg_id);
  106|      0|                        return false;
  107|      0|                    }
  108|      0|                }
  109|      0|            }
  110|       |        }
  111|       |
  112|      0|        if self.last_msg == 0 || rollover_gt(msg_id, self.last_msg) {
  113|      0|            let sub_hdr = Header::from_string("subscription", self.id.to_string());
  114|      0|            let ack_hdr = Header::from_string("ack", msg_id.to_string());
  115|      0|            let session = self.session.read().unwrap();
  116|      0|            if self.filter_self {
  117|      0|                if let Some(session_id) = message.session_id {
  118|      0|                    if session_id == session.id() {
  119|      0|                        debug!("not sending to self {}", msg_id);
  120|      0|                        return false;
  121|      0|                    }
  122|      0|                }
  123|      0|            }
  124|      0|            session.send_message_w_hdrs(message.clone(), vec![sub_hdr, ack_hdr]);
  125|      0|            self.last_msg = msg_id;
  126|      0|            debug!("published message id {}", msg_id);
  127|      0|            match self.ack {
  128|      0|                Ack::Auto => {
  129|      0|                    return true;
  130|       |                }
  131|       |                _ => {
  132|      0|                    session.pending_ack(msg_id, dest_id);
  133|      0|                    return false;
  134|       |                }
  135|       |            }
  136|       |        } else {
  137|      0|            debug!("already published {}", msg_id);
  138|      0|            return false;
  139|       |        }
  140|      0|    }
  141|       |
  142|      0|    pub fn as_string(&self) -> String {
  143|      0|        let mut debug = String::new();
  144|      0|        debug.push_str("sub");
  145|      0|
  146|      0|        let session = self.session.read().unwrap();
  147|      0|        if let Some(user) = &session.user {
  148|      0|            debug.push_str(" user=\"");
  149|      0|            debug.push_str(user.as_str());
  150|      0|            debug.push_str("\"");
  151|      0|        }
  152|      0|        debug.push_str(" created=");
  153|      0|        debug.push_str(&self.timestamp.to_rfc3339());
  154|      0|
  155|      0|        debug
  156|      0|    }
  157|       |}
  158|       |
  159|       |impl Drop for Subscription {
  160|       |    fn drop(&mut self) {
  161|       |        debug!("dropped");
  162|       |    }
  163|       |}

src/system/limits.rs:
    1|       |use libc::rlimit;
    2|       |use libc::setrlimit;
    3|       |use libc::RLIMIT_NOFILE;
    4|       |
    5|       |/// Set system limits on number of "open files", two files are used
    6|       |/// per TCP socket in Linux
    7|      1|pub fn increase_rlimit_nofile(limit: u64) -> i32 {
    8|      1|    let lim = rlimit {
    9|      1|        rlim_cur: limit,
   10|      1|        rlim_max: limit,
   11|      1|    };
   12|      1|    unsafe {
   13|      1|        return setrlimit(RLIMIT_NOFILE, &lim);
   14|      1|    }
   15|      1|}

src/util.rs:
    1|       |//! Utility methods.
    2|       |
    3|       |/// returns Some(String::from(s))
    4|      0|pub fn some_string(s: &'static str) -> Option<String> {
    5|      0|    Some(String::from(s))
    6|      0|}
    7|       |
    8|       |static ROLLOVER_LOWER_LIMIT: usize = usize::max_value() / 4;
    9|       |static ROLLOVER_UPPER_LIMIT: usize = usize::max_value() / 4 * 3;
   10|       |
   11|       |/// returns true if new > old, if we rollover usize this still works with range (1 quarter of usize)
   12|      8|pub fn rollover_gt(new: usize, old: usize) -> bool {
   13|      8|    if new < ROLLOVER_LOWER_LIMIT && old > ROLLOVER_UPPER_LIMIT {
   14|      3|        return true;
   15|       |    } else {
   16|      5|        return new > old;
   17|       |    }
   18|      8|}
   19|       |
   20|       |#[cfg(test)]
   21|       |mod tests {
   22|       |    use super::*;
   23|       |
   24|      1|    #[test]
   25|      1|    fn test_rollover_gt() {
   26|      1|        assert!(!rollover_gt(0, 0));
   27|      1|        assert!(!rollover_gt(1, 1));
   28|      1|        assert!(rollover_gt(1, 0));
   29|      1|        assert!(rollover_gt(2, 0));
   30|      1|        assert!(rollover_gt(2, 1));
   31|      1|        assert!(rollover_gt(0, usize::max_value() - 10));
   32|      1|        assert!(rollover_gt(0, usize::max_value()));
   33|      1|        assert!(rollover_gt(1, usize::max_value()));
   34|      1|    }
   35|       |}

src/web_socket/ws_demunge.rs:
    1|       |use log::Level::Debug;
    2|       |use log::*;
    3|       |
    4|       |/// Unfuck a WebSockets stream. Websockets, instead of being a simple byte stream, have pointless framing and
    5|       |/// the inputs stream is XORed with a key specifically to break proxies that don't understand the protocol.
    6|       |/// This serves no purpose, it simply complicates matters, since proxies are borken anyway and [0,0,0,0] is a valid key.
    7|       |/// Since the protocol only adds noise and never compresses data we can parse the incoming bytes and convert the input
    8|       |/// buffer back to normal plain text STOMP protocol.
    9|       |///
   10|       |/// see https://tools.ietf.org/html/rfc6455
   11|       |///
   12|       |/// Code ported from xtomp
   13|       |
   14|       |const MAX_FRAME_LEN: i64 = 1048576; // 1 Mb;
   15|       |
   16|       |/// Websockets parser state.  There are a few bytes header at the start of each frame, since these are discarded from the byte stream
   17|       |/// this struct maintains their state, its reusable, as a frame is finished the struct can be zeroed.
   18|      1|#[derive(Debug)]
   19|       |pub struct WsState {
   20|       |    opcode: u8,
   21|       |    masking_key: [u8; 4],
   22|       |    frame_len: usize,
   23|       |    frame_pos: usize,
   24|       |    frame_state: WsParserState,
   25|       |    frame_len_idx: u8,
   26|       |    frame_len_data: i64, // MUST be 64 bits for we parsing to work value is always < MAX_FRAME_LENGTH
   27|       |    masking_key_len_idx: usize,
   28|       |}
   29|       |
   30|       |impl WsState {
   31|      5|    pub fn new() -> WsState {
   32|      5|        WsState {
   33|      5|            opcode: 0,
   34|      5|            masking_key: [0; 4],
   35|      5|            frame_len: 0,
   36|      5|            frame_pos: 0,
   37|      5|            frame_state: WsParserState::SwStart,
   38|      5|            frame_len_idx: 0,
   39|      5|            frame_len_data: 0,
   40|      5|            masking_key_len_idx: 0,
   41|      5|        }
   42|      5|    }
   43|       |
   44|      0|    pub fn frame_len(&self) -> usize {
   45|      0|        self.frame_len
   46|      0|    }
   47|       |
   48|     16|    fn reset(&mut self) {
   49|     16|        // Rust manages to be more verbose than C, memset(ws, 0, sizeof(ws));
   50|     16|        self.opcode = 0;
   51|     16|        self.masking_key = [0; 4];
   52|     16|        self.frame_len = 0;
   53|     16|        self.frame_pos = 0;
   54|     16|        self.frame_state = WsParserState::SwStart;
   55|     16|        self.frame_len_idx = 0;
   56|     16|        self.frame_len_data = 0;
   57|     16|        self.masking_key_len_idx = 0;
   58|     16|    }
   59|       |}
   60|       |
   61|       |/// skip over bytes by moving data backwards
   62|     15|fn ws_skip(buffer: &mut [u8], pos: usize, end: usize, num: usize) {
   63|     15|    //debug!("before ws_skip({:?}), pos={}, num={}", String::from_utf8_lossy(&buffer), pos, num);
   64|     15|    buffer.copy_within(pos..end, pos - num);
   65|     15|    //debug!("after ws_skip({:?})", String::from_utf8_lossy(&buffer));
   66|     15|}
   67|       |
   68|       |/// do the XOR part
   69|       |/// // returns the position at the end of unmunged data or frame end
   70|     17|fn ws_demunge_buffer(ws: &mut WsState, buffer: &mut [u8], pos: usize, end: usize) -> usize {
   71|     17|    // TODO XOR the data in 64bit chunks will probably be faster. Rust support overloading ^
   72|     17|    let mut p = pos;
   73|    259|    while p < end && ws.frame_pos < ws.frame_len {
   74|    242|        buffer[p] = buffer[p] ^ ws.masking_key[ws.frame_pos % 4];
   75|    242|        p += 1;
   76|    242|        ws.frame_pos += 1;
   77|    242|    }
   78|     17|    return p;
   79|     17|}
   80|       |
   81|       |/// unmunge one frames worth of data, or all that is available now.
   82|       |/// @return number of bytes unmunged, or error if we don't like what we see when its unmunged
   83|       |/// This method enforces one frame per STOMP message, which is not strictly necessary
   84|     17|fn ws_demunge_frame(
   85|     17|    ws: &mut WsState,
   86|     17|    buffer: &mut [u8],
   87|     17|    pos: usize,
   88|     17|    end: usize,
   89|     17|    frame_lengths: &mut Box<Vec<usize>>,
   90|     17|) -> Result<usize, ()> {
   91|     17|    //debug!("ws_demunge_frame({:?}, {}, {}), frame_pos={}, frame_len={} masking_key={:?}", String::from_utf8_lossy(&buffer[pos..end]), pos, end, ws.frame_pos, ws.frame_len, ws.masking_key.as_ref());
   92|     17|
   93|     17|    let p = ws_demunge_buffer(ws, buffer, pos, end);
   94|     17|
   95|     17|    // if we reached the end of a frame
   96|     17|    if ws.frame_pos == ws.frame_len {
   97|       |        // if this is a FIN text frame
   98|     14|        if ws.opcode == 0x81 {
   99|       |            // if we did process at least one char (so p-1 is valid)
  100|       |            // and the end char of the ws frame is not '\0', i.e a valid STOMP frame, or '\n' a heart-beat
  101|       |            // fail fast.
  102|     11|            if p != pos {
  103|     11|                if ws.frame_len == 1 && buffer[p - 1] == b'\n' {
  104|       |                    // heart beat
  105|      0|                    debug!("ws ecg");
  106|     11|                } else if buffer[p - 1] != b'\0' {
  107|      0|                    warn!("ws not a stomp message, ends with '{}'", buffer[p - 1]);
  108|      0|                    if log_enabled!(Debug) {
  109|      0|                        debug!(
  110|      0|                            "ws not a stomp message buffer={:?} {:?}",
  111|      0|                            buffer,
  112|      0|                            String::from_utf8_lossy(&buffer[pos..end])
  113|       |                        );
  114|      0|                    }
  115|      0|                    return Err(());
  116|     11|                } else {
  117|     11|                    frame_lengths.push(ws.frame_len);
  118|     11|                }
  119|      0|            }
  120|      3|        }
  121|      3|    }
  122|       |
  123|     17|    return Ok(p - pos);
  124|     17|}
  125|       |
  126|     24|#[derive(Debug, Clone, PartialEq)]
  127|       |pub enum WsParserState {
  128|       |    SwStart = 0,
  129|       |    SwPayloadLen1,
  130|       |    SwPayloadLenExt,
  131|       |    SwMaskingKey,
  132|       |    SwPayload,
  133|       |    SwPayloadSkip,
  134|       |}
  135|       |
  136|       |/// We are passed the buffer containing the data from the network, with pos and end marking new data just arrived.
  137|       |/// This method hacks at the data buffer "unmunging" it and returns new pos and end, the buffer between pos and end will have usually
  138|       |/// had its data modified and/or skipped over.
  139|       |/// `buffer` may contain nothing, partial frames, a whole frame or more than one frame, all of it must be processed in one call
  140|       |// TODO This code is ported directly from xtomp, a tokio Decoder might be more appropriate?
  141|     24|pub fn ws_demunge(
  142|     24|    ws: &mut WsState,
  143|     24|    buffer: &mut Box<[u8]>,
  144|     24|    pos: usize,
  145|     24|    mut end: usize,
  146|     24|    mut frame_lengths: &mut Box<Vec<usize>>,
  147|     24|) -> Result<(usize, usize), ()> {
  148|     24|    //debug!("ws_demunge() pos={}, end={}", pos, end);
  149|     24|
  150|     24|    // flags to detect that one whole frame is delivered and nothing more
  151|     24|    let mut is_single_frame = 0;
  152|     24|    let mut p: usize;
  153|     24|    let mut ch: u8;
  154|     24|    let mut opcode: u8;
  155|     24|    let mut skipped: usize; // number of ws frame info bytes read
  156|     24|    let mut processed: usize;
  157|     24|
  158|     24|    let mut state = ws.frame_state.clone();
  159|     24|    skipped = 0;
  160|     24|
  161|     24|    if pos == 0 {
  162|     23|        is_single_frame += 1;
  163|     23|    }
  164|       |
  165|     24|    p = pos;
  166|    151|    while p < end {
  167|       |        // TODO off by one? should it be <= ?
  168|    129|        ch = buffer[p];
  169|    129|
  170|    129|        match state {
  171|    129|            //* REQUEST command */
  172|    129|            WsParserState::SwStart => {
  173|     16|                ws.reset();
  174|     16|                ws.opcode = ch;
  175|     16|
  176|     16|                // FIN bit set
  177|     16|                if (ch & 0x80) == 0x80 {
  178|     15|                    is_single_frame += 1;
  179|     15|                }
  180|     16|                state = WsParserState::SwPayloadLen1;
  181|     16|
  182|     16|                skipped += 1;
  183|     16|                p += 1;
  184|     16|                continue;
  185|       |            }
  186|       |            WsParserState::SwPayloadLen1 => {
  187|     16|                if (ch & 0x80) != 0x80 {
  188|      0|                    warn!("BUG client data must be masked");
  189|      0|                    return Err(());
  190|     16|                }
  191|     16|
  192|     16|                if (ch & 0x7f) == 126 {
  193|      0|                    ws.frame_len_idx = 2;
  194|      0|                    state = WsParserState::SwPayloadLenExt;
  195|     16|                } else if (ch & 0x7f) == 127 {
  196|      2|                    ws.frame_len_idx = 8;
  197|      2|                    state = WsParserState::SwPayloadLenExt;
  198|     14|                } else {
  199|     14|                    ws.frame_len = (ch & 0x7f) as usize;
  200|     14|                    state = WsParserState::SwMaskingKey;
  201|     14|                }
  202|       |
  203|     16|                skipped += 1;
  204|     16|                p += 1;
  205|     16|                continue;
  206|       |            }
  207|       |            WsParserState::SwPayloadLenExt => {
  208|       |                // shift 64bit arriving as u8 bytes
  209|     16|                ws.frame_len_idx -= 1;
  210|     16|                ws.frame_len_data |= (ch as i64) << (8 * ws.frame_len_idx) as i64;
  211|     16|
  212|     16|                if ws.frame_len_idx == 0 {
  213|      2|                    if ws.frame_len_data < 0 || ws.frame_len_data > MAX_FRAME_LEN {
  214|      0|                        debug!("ws frame flup {}", ws.frame_len_data);
  215|      0|                        return Err(());
  216|      2|                    }
  217|      2|                    ws.frame_len = ws.frame_len_data as usize;
  218|      2|                    state = WsParserState::SwMaskingKey;
  219|     14|                }
  220|       |
  221|     16|                skipped += 1;
  222|     16|                p += 1;
  223|     16|
  224|     16|                continue;
  225|       |            }
  226|       |            WsParserState::SwMaskingKey => {
  227|     64|                ws.masking_key[ws.masking_key_len_idx] = ch;
  228|     64|                ws.masking_key_len_idx += 1;
  229|     64|
  230|     64|                if ws.masking_key_len_idx == 4 {
  231|     16|                    state = WsParserState::SwPayload;
  232|     48|                }
  233|       |
  234|     64|                skipped += 1;
  235|     64|                p += 1;
  236|     64|                continue;
  237|       |            }
  238|       |            WsParserState::SwPayload => {
  239|       |                // TODO why do this here?
  240|     17|                if ws.frame_len > MAX_FRAME_LEN as usize {
  241|      0|                    debug!("ws frame flup {}", ws.frame_len);
  242|      0|                    return Err(());
  243|     17|                }
  244|     17|
  245|     17|                opcode = ws.opcode & 0x0f;
  246|     17|                if opcode == 0x08 {
  247|       |                    // connection close
  248|      0|                    warn!("ws connection close");
  249|      0|                    return Err(());
  250|     17|                }
  251|     17|                if opcode == 0x02 {
  252|       |                    // binary not supported
  253|      0|                    warn!("ws binary not supported");
  254|      0|                    return Err(());
  255|     17|                }
  256|     17|
  257|     17|                match ws_demunge_frame(ws, buffer, p, end, &mut frame_lengths) {
  258|     17|                    Ok(sz) => {
  259|     17|                        processed = sz;
  260|     17|                    }
  261|       |                    Err(_) => {
  262|      0|                        warn!("ws_demunge_frame() err");
  263|      0|                        return Err(());
  264|       |                    }
  265|       |                }
  266|       |
  267|     17|                if opcode == 0x09 || opcode == 0x0a {
  268|       |                    // per stackoverflow https://stackoverflow.com/questions/10585355/sending-websocket-ping-pong-frame-from-browser
  269|       |                    // clients dont send pings, so we dont write complicated code to handle them
  270|       |                    // however just in case, we will skip over them.
  271|      1|                    debug!("unhandled ws ping");
  272|     16|                }
  273|       |
  274|       |                // any unsupported opcodes, skip over.
  275|     17|                if opcode > 0x02 {
  276|      1|                    debug!("unknown opcode skip processed data");
  277|       |                    // shift buffer erasing ws junk
  278|      1|                    ws_skip(buffer, p + processed, end, processed + skipped);
  279|      1|                    p -= skipped;
  280|      1|                    end -= processed + skipped;
  281|      1|                    //debug!("p={}, skipped={} ", p, skipped);
  282|      1|                    skipped = 0;
  283|      1|                    is_single_frame = 0;
  284|      1|                    if ws.frame_pos == ws.frame_len {
  285|      1|                        state = WsParserState::SwStart;
  286|      1|                    } else if ws.frame_pos < ws.frame_len {
  287|      0|                        state = WsParserState::SwPayloadSkip;
  288|      0|                    } else {
  289|      0|                        error!("BUG overread ws buf");
  290|      0|                        return Err(());
  291|       |                    }
  292|      1|                    continue;
  293|     16|                }
  294|     16|
  295|     16|                // now we have read all of the ws header and it can be discarded
  296|     16|
  297|     16|                if is_single_frame == 2 && buffer.len() - pos - skipped == ws.frame_len {
  298|       |                    // optimization: instead of moving bytes in the buffer, just shift pos, can only do this when we have exactly one frame
  299|       |                    // just one frame, is the most common case.
  300|      2|                    ws.frame_state = WsParserState::SwStart;
  301|      2|                    return Ok((pos + skipped, end));
  302|       |                } else {
  303|       |                    // shift buffer erasing ws junk
  304|       |                    //debug!("skipping header");
  305|     14|                    ws_skip(buffer, p, end, skipped);
  306|     14|                    p = p - skipped + processed;
  307|     14|                    end -= skipped;
  308|     14|                    skipped = 0;
  309|     14|                    is_single_frame = 0;
  310|     14|                    if ws.frame_pos == ws.frame_len {
  311|     11|                        state = WsParserState::SwStart;
  312|     11|                    }
  313|       |                    //debug!("looping {} {}", p, end);
  314|     14|                    continue;
  315|       |                }
  316|       |            }
  317|       |            WsParserState::SwPayloadSkip => {
  318|      0|                match ws_demunge_frame(ws, buffer, p, end, &mut frame_lengths) {
  319|      0|                    Ok(sz) => {
  320|      0|                        processed = sz;
  321|      0|                    }
  322|       |                    Err(_) => {
  323|      0|                        warn!("ws_demunge_frame() error");
  324|      0|                        return Err(());
  325|       |                    }
  326|       |                }
  327|      0|                ws_skip(buffer, p, end, processed);
  328|      0|                p = p - processed;
  329|      0|                end -= processed;
  330|      0|                if ws.frame_pos == ws.frame_len {
  331|      0|                    state = WsParserState::SwStart;
  332|      0|                }
  333|      0|                continue;
  334|       |            }
  335|       |        }
  336|       |    }
  337|       |
  338|     22|    ws.frame_state = state;
  339|     22|
  340|     22|    if p == pos {
  341|       |        // did not read anything but everything is OK
  342|      0|        return Ok((pos, end));
  343|     22|    }
  344|     22|
  345|     22|    return Ok((pos + skipped, end));
  346|     24|}
  347|       |
  348|       |#[cfg(test)]
  349|       |mod tests {
  350|       |    use super::*;
  351|       |
  352|       |    const LEN: usize = 20;
  353|       |
  354|     12|    fn setup_frame_hdr(buffer: &mut Box<[u8]>, len: usize) {
  355|     12|        // opcodes (FIN and text data)
  356|     12|        buffer[0] = 0x81;
  357|     12|        // payload len
  358|     12|        buffer[1] = len as u8 | 0x80; // less than 126
  359|     12|                                      // noop mask
  360|     12|        buffer[2] = 0x00;
  361|     12|        buffer[3] = 0x00;
  362|     12|        buffer[4] = 0x00;
  363|     12|        buffer[5] = 0x00;
  364|     12|    }
  365|       |
  366|     11|    fn setup_frame_data(buffer: &mut Box<[u8]>) {
  367|     11|        // mocked data "ghijklmnopqrstuvwxy\0" (last digit is left as \0 to appear like a STOMP frame)
  368|     11|        let mut p = 6;
  369|    220|        while p < LEN + 5 {
  370|    209|            buffer[p] = b'a' + p as u8;
  371|    209|            p += 1;
  372|    209|        }
  373|     11|    }
  374|       |
  375|      1|    fn set_mask(buffer: &mut Box<[u8]>) {
  376|      1|        buffer[2] = 1;
  377|      1|        buffer[3] = 2;
  378|      1|        buffer[4] = 3;
  379|      1|        buffer[5] = 4;
  380|      1|    }
  381|       |
  382|      1|    fn mask_frame_data(buffer: &mut Box<[u8]>) {
  383|      1|        let mut p = 6;
  384|      1|        let mut m = 1;
  385|     21|        while p < LEN + 6 {
  386|     20|            buffer[p] = buffer[p] ^ m;
  387|     20|            p += 1;
  388|     20|            m += 1;
  389|     20|            if m > 4 {
  390|      5|                m = 1;
  391|     15|            }
  392|       |        }
  393|      1|    }
  394|       |
  395|      1|    #[test]
  396|      1|    fn test_happy_noop_mask() {
  397|      1|        //config::log::init_log4rs();
  398|      1|
  399|      1|        let mut frame_lengths = Box::new(vec![]);
  400|      1|
  401|      1|        let mut buffer: Box<[u8]> = Box::new([0; 6 + LEN]);
  402|      1|        let pos: usize = 0;
  403|      1|        let end = buffer.len();
  404|      1|
  405|      1|        setup_frame_hdr(&mut buffer, LEN);
  406|      1|        setup_frame_data(&mut buffer);
  407|      1|
  408|      1|        let mut ws = WsState::new();
  409|      1|
  410|      1|        match ws_demunge(&mut ws, &mut buffer, pos, end, &mut frame_lengths) {
  411|      1|            Ok((new_pos, new_end)) => {
  412|      1|                println!(
  413|      1|                    "pos={}, end={} buffer={:?}",
  414|      1|                    new_pos,
  415|      1|                    new_end,
  416|      1|                    String::from_utf8_lossy(&buffer[new_pos..new_end])
  417|      1|                );
  418|      1|                // pos is shifted 6 forward since this was only one whole frame, buffer is not changed and pos is adjusted
  419|      1|                assert_eq!(pos + 6, new_pos, "new_pos test");
  420|      1|                assert_eq!(LEN, new_end - new_pos, "len test");
  421|      1|                assert_eq!(b"ghijklmnopqrstuvwxy\0", &buffer[new_pos..new_end]);
  422|       |            }
  423|      0|            _ => panic!("parse error"),
  424|       |        }
  425|       |
  426|       |        // reset the test data
  427|      1|        setup_frame_hdr(&mut buffer, LEN);
  428|      1|        setup_frame_data(&mut buffer);
  429|      1|
  430|      1|        // arbitrary chop point  16
  431|      1|
  432|      1|        let mut buffer1: Box<[u8]> = Box::new([0; 16]);
  433|      1|        buffer1.copy_from_slice(&buffer[0..16]);
  434|      1|        let pos1 = 0;
  435|      1|        let end1 = buffer1.len();
  436|      1|
  437|      1|        let mut buffer2: Box<[u8]> = Box::new([0; 10]);
  438|      1|        buffer2.copy_from_slice(&buffer[16..26]);
  439|      1|        let pos2 = 0;
  440|      1|        let end2 = buffer2.len();
  441|      1|
  442|      1|        match ws_demunge(&mut ws, &mut buffer1, pos1, end1, &mut frame_lengths) {
  443|      1|            Ok((new_pos, new_end)) => {
  444|      1|                // pos is left at 0, the data in buffer is shifted forward
  445|      1|                assert_eq!(0, new_pos);
  446|      1|                assert_eq!(10, new_end);
  447|       |            }
  448|      0|            _ => panic!("parse error"),
  449|       |        }
  450|      1|        match ws_demunge(&mut ws, &mut buffer2, pos2, end2, &mut frame_lengths) {
  451|      1|            Ok((new_pos, new_end)) => {
  452|      1|                // buffer not changed at all
  453|      1|                println!(
  454|      1|                    "new_pos={}, new_end={} buffer={:?}",
  455|      1|                    new_pos,
  456|      1|                    new_end,
  457|      1|                    String::from_utf8_lossy(&buffer2[0..10])
  458|      1|                );
  459|      1|                println!("WsState {:?}", ws)
  460|       |            }
  461|      0|            _ => panic!("parse error"),
  462|       |        }
  463|       |
  464|       |        // lots of different chop points, just check it does not panic, TODO assert logic
  465|       |
  466|      1|        setup_frame_hdr(&mut buffer, LEN);
  467|      1|        setup_frame_data(&mut buffer);
  468|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 1);
  469|      1|
  470|      1|        setup_frame_hdr(&mut buffer, LEN);
  471|      1|        setup_frame_data(&mut buffer);
  472|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 2);
  473|      1|
  474|      1|        setup_frame_hdr(&mut buffer, LEN);
  475|      1|        setup_frame_data(&mut buffer);
  476|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 3);
  477|      1|
  478|      1|        setup_frame_hdr(&mut buffer, LEN);
  479|      1|        setup_frame_data(&mut buffer);
  480|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 4);
  481|      1|
  482|      1|        setup_frame_hdr(&mut buffer, LEN);
  483|      1|        setup_frame_data(&mut buffer);
  484|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 5);
  485|      1|
  486|      1|        setup_frame_hdr(&mut buffer, LEN);
  487|      1|        setup_frame_data(&mut buffer);
  488|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 6);
  489|      1|
  490|      1|        setup_frame_hdr(&mut buffer, LEN);
  491|      1|        setup_frame_data(&mut buffer);
  492|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 7);
  493|      1|
  494|      1|        setup_frame_hdr(&mut buffer, LEN);
  495|      1|        setup_frame_data(&mut buffer);
  496|      1|        re_test(&mut ws, buffer.clone(), 0, 26, 8);
  497|      1|    }
  498|       |
  499|      8|    fn re_test(mut ws: &mut WsState, buffer: Box<[u8]>, _pos: usize, _len: usize, split_at: usize) {
  500|      8|        let mut frame_lengths = Box::new(vec![]);
  501|      8|
  502|      8|        let mut buffer1: Box<[u8]> = vec![0; split_at].into_boxed_slice();
  503|      8|        buffer1.copy_from_slice(&buffer[0..split_at]);
  504|      8|        let pos1 = 0;
  505|      8|        let end1 = buffer1.len();
  506|      8|
  507|      8|        let mut buffer2: Box<[u8]> = vec![0; buffer.len() - split_at].into_boxed_slice();
  508|      8|        buffer2.copy_from_slice(&buffer[split_at..buffer.len()]);
  509|      8|        let pos2 = 0;
  510|      8|        let end2 = buffer2.len();
  511|      8|
  512|      8|        match ws_demunge(&mut ws, &mut buffer1, pos1, end1, &mut frame_lengths) {
  513|      8|            Ok((_new_pos, _new_end)) => {}
  514|      0|            _ => panic!("parse error"),
  515|       |        }
  516|      8|        match ws_demunge(&mut ws, &mut buffer2, pos2, end2, &mut frame_lengths) {
  517|      8|            Ok((_new_pos, _new_end)) => {}
  518|      0|            _ => panic!("parse error"),
  519|       |        }
  520|      8|    }
  521|       |
  522|      1|    #[test]
  523|      1|    fn test_happy_with_mask() {
  524|      1|        //config::log::init_log4rs();
  525|      1|
  526|      1|        let mut frame_lengths = Box::new(vec![]);
  527|      1|
  528|      1|        let mut buffer: Box<[u8]> = Box::new([0; 6 + LEN]);
  529|      1|        let pos: usize = 0;
  530|      1|        let end = buffer.len();
  531|      1|
  532|      1|        setup_frame_hdr(&mut buffer, LEN);
  533|      1|        setup_frame_data(&mut buffer);
  534|      1|        set_mask(&mut buffer);
  535|      1|        mask_frame_data(&mut buffer);
  536|      1|
  537|      1|        let mut ws = WsState::new();
  538|      1|
  539|      1|        match ws_demunge(&mut ws, &mut buffer, pos, end, &mut frame_lengths) {
  540|      1|            Ok((new_pos, new_end)) => {
  541|      1|                println!(
  542|      1|                    "pos={}, end={} buffer={:?}",
  543|      1|                    new_pos,
  544|      1|                    new_end,
  545|      1|                    String::from_utf8_lossy(&buffer[new_pos..new_end])
  546|      1|                );
  547|      1|                assert_eq!(LEN, new_end - new_pos, "len test");
  548|      1|                assert_eq!(pos + 6, new_pos, "new_pos test");
  549|      1|                assert_eq!(b"ghijklmnopqrstuvwxy\0", &buffer[new_pos..new_end]);
  550|       |            }
  551|      0|            _ => panic!("parse error"),
  552|       |        }
  553|      1|    }
  554|       |
  555|      1|    #[test]
  556|      1|    fn test_skip_ping_frames() {
  557|      1|        //config::log::init_log4rs();
  558|      1|
  559|      1|        let mut frame_lengths = Box::new(vec![]);
  560|      1|
  561|      1|        let mut buffer: Box<[u8]> = Box::new([0; 6 + 4 + 8 + 6 + 16]);
  562|      1|        let pos: usize = 0;
  563|      1|        let end = buffer.len();
  564|      1|
  565|      1|        let prefix = "CONN";
  566|      1|        setup_frame_hdr(&mut buffer, prefix.len());
  567|      1|        // hack initial frame to be non final
  568|      1|        buffer[0] = 0x01;
  569|      1|        buffer[6..6 + 4].copy_from_slice(prefix.as_bytes());
  570|      1|
  571|      1|        // ping frame
  572|      1|        buffer[10] = 0x89; // FIN + op 9 ping
  573|      1|        buffer[11] = 0x82; // mask plus len 2
  574|      1|        buffer[12] = 0x00; // noop mask
  575|      1|        buffer[13] = 0x00;
  576|      1|        buffer[14] = 0x00;
  577|      1|        buffer[15] = 0x00;
  578|      1|        buffer[16] = 0xaa; // 2 bytes data
  579|      1|        buffer[17] = 0xbb;
  580|      1|
  581|      1|        // next frame
  582|      1|        let suffix = "ECT\n1234:1234\n\n\0";
  583|      1|        // opcodes (continuation frame and text data)
  584|      1|        buffer[18] = 0x80;
  585|      1|        // payload len
  586|      1|        buffer[19] = suffix.len() as u8 | 0x80; // less than 126
  587|      1|                                                // noop mask
  588|      1|        buffer[20] = 0x00;
  589|      1|        buffer[21] = 0x00;
  590|      1|        buffer[22] = 0x00;
  591|      1|        buffer[23] = 0x00;
  592|      1|        buffer[24..end].copy_from_slice(suffix.as_bytes());
  593|      1|
  594|      1|        let mut ws = WsState::new();
  595|      1|
  596|      1|        match ws_demunge(&mut ws, &mut buffer, pos, end, &mut frame_lengths) {
  597|      1|            Ok((new_pos, new_end)) => {
  598|      1|                println!(
  599|      1|                    "pos={}, end={} buffer={:?}",
  600|      1|                    new_pos,
  601|      1|                    new_end,
  602|      1|                    String::from_utf8_lossy(&buffer[new_pos..new_end])
  603|      1|                );
  604|      1|                assert_eq!(prefix.len() + suffix.len(), new_end - new_pos, "len test");
  605|      1|                assert_eq!(0, new_pos, "new_pos test");
  606|       |            }
  607|      0|            _ => panic!("parse error"),
  608|       |        }
  609|      1|    }
  610|       |
  611|      1|    #[test]
  612|      1|    pub fn test_mask() {
  613|      1|        if (0xaa & 0x80) != 0x80 {
  614|       |            panic!("here");
  615|      1|        }
  616|      1|        println!("OK");
  617|      1|    }
  618|       |
  619|       |    // real data from a FF packet this sends JUST the frame header and zero STONP µessage data.
  620|      1|    #[test]
  621|      1|    pub fn test_firefox_media_header() {
  622|      1|        let mut frame_lengths = Box::new(vec![]);
  623|      1|        let mut ws = WsState::new();
  624|      1|        let mut buffer: Box<[u8]> = Box::new([0; 14]);
  625|      1|        buffer[0] = 129; // 1 0 0 0  0 0 0 1  =  0x81 fin text
  626|      1|        buffer[1] = 255; // 1 1 1 1  1 1 1 1  = mask + 127 len
  627|      1|        buffer[2] = 0;
  628|      1|        buffer[3] = 0;
  629|      1|        buffer[4] = 0;
  630|      1|        buffer[5] = 0;
  631|      1|        buffer[6] = 0;
  632|      1|        buffer[7] = 6; // 0000 0110
  633|      1|        buffer[8] = 253; //
  634|      1|        buffer[9] = 129; //
  635|      1|        buffer[10] = 212; // key
  636|      1|        buffer[11] = 137; // key
  637|      1|        buffer[12] = 3; // key
  638|      1|        buffer[13] = 53; // key
  639|      1|
  640|      1|        match ws_demunge(&mut ws, &mut buffer, 0, 14, &mut frame_lengths) {
  641|      1|            Ok((new_pos, new_end)) => {
  642|      1|                println!(
  643|      1|                    "pos={}, end={}, state={:?}",
  644|      1|                    new_pos, new_end, ws.frame_state
  645|      1|                );
  646|      1|                assert_eq!(new_pos, 14); // read all the frame data but no STOMP data
  647|      1|                assert_eq!(new_end, 14);
  648|      1|                assert_eq!(ws.frame_len, 458113);
  649|       |            }
  650|      0|            _ => panic!("parse error"),
  651|       |        }
  652|      1|    }
  653|       |
  654|       |    // same test but imagine we only got opart of the frame header
  655|      1|    #[test]
  656|      1|    pub fn test_firefox_media_header_partial() {
  657|      1|        let mut frame_lengths = Box::new(vec![]);
  658|      1|        let mut ws = WsState::new();
  659|      1|        let mut buffer: Box<[u8]> = Box::new([0; 7]);
  660|      1|        buffer[0] = 129; // 1 0 0 0  0 0 0 1  =  0x81 fin text
  661|      1|        buffer[1] = 255; // 1 1 1 1  1 1 1 1  = mask + 127 len
  662|      1|        buffer[2] = 0;
  663|      1|        buffer[3] = 0;
  664|      1|        buffer[4] = 0;
  665|      1|        buffer[5] = 0;
  666|      1|        buffer[6] = 0;
  667|      1|
  668|      1|        match ws_demunge(&mut ws, &mut buffer, 0, 7, &mut frame_lengths) {
  669|      1|            Ok((new_pos, new_end)) => {
  670|      1|                println!(
  671|      1|                    "pos={}, end={}, state={:?}",
  672|      1|                    new_pos, new_end, ws.frame_state
  673|      1|                );
  674|      1|                assert_eq!(new_pos, 7); // read all the frame data but no STOMP data
  675|      1|                assert_eq!(new_end, 7);
  676|       |            }
  677|      0|            _ => panic!("parse error"),
  678|       |        }
  679|       |        // no read lets read the rest
  680|      1|        let mut buffer: Box<[u8]> = Box::new([0; 14]);
  681|      1|        buffer[7] = 6; // 0000 0110
  682|      1|        buffer[8] = 253; //
  683|      1|        buffer[9] = 129; //
  684|      1|        buffer[10] = 212; // key
  685|      1|        buffer[11] = 137; // key
  686|      1|        buffer[12] = 3; // key
  687|      1|        buffer[13] = 53; // key
  688|      1|
  689|      1|        match ws_demunge(&mut ws, &mut buffer, 7, 14, &mut frame_lengths) {
  690|      1|            Ok((new_pos, new_end)) => {
  691|      1|                println!(
  692|      1|                    "pos={}, end={}, state={:?}",
  693|      1|                    new_pos, new_end, ws.frame_state
  694|      1|                );
  695|      1|                assert_eq!(new_pos, 14); // read all the frame data but no STOMP data
  696|      1|                assert_eq!(new_end, 14);
  697|      1|                assert_eq!(ws.frame_len, 458113);
  698|       |            }
  699|      0|            _ => panic!("parse error"),
  700|       |        }
  701|      1|    }
  702|       |}

src/web_socket/ws_headers.rs:
    1|       |//! Code for validating a WebSockets Upgrade HTTP request.
    2|       |//!
    3|       |//! <pre>
    4|       |//! GET /foo?ba=true HTTP/1.1
    5|       |//! Host: xtomp.tp23.org
    6|       |//! ignore: me
    7|       |//! Upgrade: websocket
    8|       |//! Connection: Upgrade
    9|       |//! Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
   10|       |//! Sec-WebSocket-Protocol: stomp
   11|       |//! Sec-WebSocket-Version: 13
   12|       |//! Origin: http://tp23.org
   13|       |//! </pre>
   14|       |//!
   15|       |use log::Level::Debug;
   16|       |use log::*;
   17|       |
   18|       |use sha1::Sha1;
   19|       |
   20|       |use crate::init::CONFIG;
   21|       |use crate::message::stomp_message::StompMessage;
   22|       |
   23|      0|#[derive(Debug, PartialEq)]
   24|       |pub enum WsUpgradeError {
   25|       |    OriginDenied,
   26|       |    Syntax,
   27|       |    HostMissing,
   28|       |    ProtocolError,
   29|       |}
   30|       |
   31|       |/// Validate all the HTTP headers required to upgrade to WebSockets, spec writers had a field day here!
   32|       |/// All that is really necessary is  `GET / HTTP/1.1\nUpgrade:websocket\n\n`
   33|      4|pub fn ws_validate_hdrs(message: &StompMessage) -> Result<(), WsUpgradeError> {
   34|      4|    //    if let Stomp = &message.message_type {
   35|      4|    //        return Err(WsUpgradeError::Syntax);
   36|      4|    //    }
   37|      4|    if !ws_validate_hdr_websocket_origin(message) {
   38|      0|        return Err(WsUpgradeError::OriginDenied);
   39|      4|    }
   40|      4|    if !ws_validate_hdr_host(message) {
   41|      1|        return Err(WsUpgradeError::HostMissing);
   42|      3|    }
   43|      3|    if !ws_validate_hdr_sec_protocol(message) {
   44|      0|        return Err(WsUpgradeError::ProtocolError);
   45|      3|    }
   46|      3|    if !ws_validate_hdr_upgrade(message) {
   47|      1|        return Err(WsUpgradeError::ProtocolError);
   48|      2|    }
   49|      2|
   50|      2|    if ws_validate_hdr_connection(message)
   51|      2|        && ws_validate_hdr_upgrade(message)
   52|      2|        && ws_validate_hdr_websocket_version(message)
   53|      2|        && ws_validate_hdr_websocket_key(message)
   54|       |    {
   55|      2|        return Ok(());
   56|      0|    }
   57|      0|
   58|      0|    Err(WsUpgradeError::Syntax)
   59|      4|}
   60|       |
   61|       |/// Get the return accept key, key should be 16 bytes base64 encoded but
   62|       |/// servers do not need to verify this or decode the string
   63|       |pub fn ws_get_websocket_accept_key(message: &StompMessage) -> String {
   64|      2|    if let Some(web_socket_key) = message.get_header_case_insensitive("Sec-WebSocket-Key") {
   65|      2|        let mut sha1 = Sha1::new();
   66|      2|        sha1.update(web_socket_key.trim().as_bytes());
   67|      2|        sha1.update(b"258EAFA5-E914-47DA-95CA-C5AB0DC85B11" as &[u8]);
   68|      2|        let sha_res = sha1.digest().bytes();
   69|      2|        return base64::encode(sha_res.as_ref());
   70|      0|    }
   71|      0|    // probably should die
   72|      0|    return String::from("");
   73|      2|}
   74|       |
   75|       |/// parse the HTTP message's request line
   76|       |// Request line of HTTP messages is saved as a header, see parser.rs request_line_done()
   77|      0|pub fn ws_parse_request_line(message: &StompMessage) -> Result<String, ()> {
   78|      0|    match message.get_header("request-line") {
   79|      0|        Some(line) => {
   80|      0|            for (i, part) in line.split_whitespace().enumerate() {
   81|      0|                if i == 0 && !part.eq("GET") {
   82|       |                    // BUG
   83|      0|                    return Err(());
   84|      0|                }
   85|      0|                if i == 2 && !part.eq("HTTP/1.1") {
   86|       |                    // TODO should we support 2.0 and 3.0?
   87|      0|                    return Err(());
   88|      0|                }
   89|      0|                if i == 1 {
   90|       |                    // part is the URI
   91|      0|                    return Ok(String::from(part));
   92|      0|                }
   93|       |            }
   94|      0|            Err(())
   95|       |        }
   96|      0|        _ => Err(()),
   97|       |    }
   98|      0|}
   99|       |
  100|       |/// Required by rfc6455 ensures we are HTTP/1.1 at least
  101|       |/// // TODO virtual hosting
  102|       |pub fn ws_validate_hdr_host(message: &StompMessage) -> bool {
  103|      4|    if let Some(host) = message.get_header_case_insensitive("Host") {
  104|      3|        if !CONFIG.name.eq(host.trim()) {
  105|      0|            if log_enabled!(Debug) {
  106|      0|                debug!("unexpected host header: {}", host.trim());
  107|      0|            }
  108|      3|        }
  109|       |        // return true anyway could be using IP address or localhost
  110|      3|        return true;
  111|      1|    }
  112|      1|    false
  113|      4|}
  114|       |
  115|       |/// Required by rfc6455 to indicate converting an HTTP stream to something else.
  116|       |fn ws_validate_hdr_upgrade(message: &StompMessage) -> bool {
  117|      5|    if let Some(value) = message.get_header_case_insensitive("Upgrade") {
  118|      4|        return "websocket".eq(value.trim());
  119|      1|    }
  120|      1|    false
  121|      5|}
  122|       |
  123|       |/// Required by rfc6455, but pointless, FireFox now sends "keep-alive, Upgrade"
  124|       |fn ws_validate_hdr_connection(message: &StompMessage) -> bool {
  125|      2|    if let Some(value) = message.get_header_case_insensitive("Connection") {
  126|      2|        for con_flag in value.split(",") {
  127|      2|            if "Upgrade".eq_ignore_ascii_case(con_flag.trim()) {
  128|      2|                return true;
  129|      0|            }
  130|       |        }
  131|      0|    }
  132|      0|    false
  133|      2|}
  134|       |
  135|       |/// Validate secondary protocol is STOMP, fail fast if something weird connects to us.
  136|       |fn ws_validate_hdr_sec_protocol(message: &StompMessage) -> bool {
  137|      3|    if let Some(protocol) = message.get_header_case_insensitive("Sec-WebSocket-Protocol") {
  138|      3|        return "stomp".eq_ignore_ascii_case(protocol.trim());
  139|      0|    }
  140|      0|    false
  141|      3|}
  142|       |
  143|       |/// Pretty pointless, but rfc6455 requires Sec-WebSocket-Version: 13 be sent
  144|       |// TODO why should we care about silly reqs like this
  145|       |fn ws_validate_hdr_websocket_version(message: &StompMessage) -> bool {
  146|      2|    if let Some(version) = message.get_header_case_insensitive("Sec-WebSocket-Version") {
  147|      2|        return "13".eq(version.trim());
  148|      0|    }
  149|      0|    false
  150|      2|}
  151|       |
  152|       |/// Validate the Origin header, browsers MUST sewnd this so we should validate it to stop
  153|       |/// injected JavaScript hitting us from insecure websites.
  154|       |/// TGhe header is optional for Apps and non-browser connections.
  155|       |fn ws_validate_hdr_websocket_origin(message: &StompMessage) -> bool {
  156|      4|    if let Some(origin) = message.get_header_case_insensitive("Origin") {
  157|      4|        if let Some(allowed_origins) = &CONFIG.websockets_origin {
  158|      4|            if "*".eq(allowed_origins) {
  159|      0|                return true;
  160|      4|            }
  161|      8|            for allowed_origin in allowed_origins.split_whitespace() {
  162|      8|                if allowed_origin.eq_ignore_ascii_case(origin) {
  163|      4|                    return true;
  164|      4|                }
  165|       |            }
  166|      0|        }
  167|      0|    }
  168|      0|    true
  169|      4|}
  170|       |
  171|       |/// validate Sec-WebSocket-Key header
  172|      2|fn ws_validate_hdr_websocket_key(message: &StompMessage) -> bool {
  173|      2|    message
  174|      2|        .get_header_case_insensitive("Sec-WebSocket-Key")
  175|      2|        .is_some()
  176|      2|}
  177|       |
  178|       |#[cfg(test)]
  179|       |mod tests {
  180|       |    use super::*;
  181|       |    use crate::message::stomp_message::MessageType::Http;
  182|       |    use crate::message::stomp_message::Ownership;
  183|       |
  184|      1|    #[test]
  185|      1|    fn test_happy_path() {
  186|      1|        let mut message = StompMessage::new(Ownership::Session);
  187|      1|        message.message_type = Http;
  188|      1|        message.add_header("request-line", "GET / HTTP/1.1");
  189|      1|        message.add_header("Host", "romp");
  190|      1|        message.add_header("Connection", "Upgrade");
  191|      1|        message.add_header("Upgrade", "websocket");
  192|      1|        message.add_header("Sec-WebSocket-Key", "x3JJHMbDL1EzLkh9GBhXDw==");
  193|      1|        message.add_header("Sec-WebSocket-Protocol", "stomp");
  194|      1|        message.add_header("Sec-WebSocket-Version", "13");
  195|      1|        message.add_header("Origin", "http://tp23.org");
  196|      1|        match ws_validate_hdrs(&message) {
  197|      1|            Ok(_) => {
  198|      1|                assert_eq!(
  199|      1|                    "HSmrc0sMlYUkAGmm5OPpG2HaGWk=",
  200|      1|                    ws_get_websocket_accept_key(&message)
  201|      1|                );
  202|       |            }
  203|      0|            _ => panic!("Headers not ok"),
  204|       |        }
  205|      1|    }
  206|       |
  207|      1|    #[test]
  208|      1|    fn test_happy_path2() {
  209|      1|        let mut message = StompMessage::new(Ownership::Session);
  210|      1|        message.message_type = Http;
  211|      1|        message.add_header("request-line", "GET / HTTP/1.1");
  212|      1|        message.add_header("Host", "romp");
  213|      1|        message.add_header("Connection", "Upgrade");
  214|      1|        message.add_header("Upgrade", "websocket");
  215|      1|        message.add_header("Sec-WebSocket-Key", "dGhlIHNhbXBsZSBub25jZQ==");
  216|      1|        message.add_header("Sec-WebSocket-Protocol", "stomp");
  217|      1|        message.add_header("Sec-WebSocket-Version", "13");
  218|      1|        message.add_header("Origin", "http://tp23.org");
  219|      1|        match ws_validate_hdrs(&message) {
  220|      1|            Ok(_) => {
  221|      1|                assert_eq!(
  222|      1|                    "s3pPLMBiTxaQ9kYGzzhZRbK+xOo=",
  223|      1|                    ws_get_websocket_accept_key(&message)
  224|      1|                );
  225|       |            }
  226|      0|            _ => panic!("Headers not ok"),
  227|       |        }
  228|      1|    }
  229|       |
  230|      1|    #[test]
  231|      1|    fn test_neg_no_host() {
  232|      1|        let mut message = StompMessage::new(Ownership::Session);
  233|      1|        message.message_type = Http;
  234|      1|        message.add_header("request-line", "GET / HTTP/1.1");
  235|      1|        //message.add_header("Host", "romp");
  236|      1|        message.add_header("Connection", "Upgrade");
  237|      1|        message.add_header("Upgrade", "websocket");
  238|      1|        message.add_header("Sec-WebSocket-Key", "dGhlIHNhbXBsZSBub25jZQ==");
  239|      1|        message.add_header("Sec-WebSocket-Protocol", "stomp");
  240|      1|        message.add_header("Sec-WebSocket-Version", "13");
  241|      1|        message.add_header("Origin", "http://tp23.org");
  242|      1|        match ws_validate_hdrs(&message) {
  243|      1|            Err(WsUpgradeError::HostMissing) => {
  244|      1|                // expected
  245|      1|            }
  246|      0|            _ => panic!("Host checking failed"),
  247|       |        }
  248|      1|    }
  249|       |
  250|      1|    #[test]
  251|      1|    fn test_neg_no_upgrade() {
  252|      1|        let mut message = StompMessage::new(Ownership::Session);
  253|      1|        message.message_type = Http;
  254|      1|        message.add_header("request-line", "GET / HTTP/1.1");
  255|      1|        message.add_header("Host", "romp");
  256|      1|        message.add_header("Connection", "Upgrade");
  257|      1|        //message.add_header("Upgrade", "websocket");
  258|      1|        message.add_header("Sec-WebSocket-Key", "dGhlIHNhbXBsZSBub25jZQ==");
  259|      1|        message.add_header("Sec-WebSocket-Protocol", "stomp");
  260|      1|        message.add_header("Sec-WebSocket-Version", "13");
  261|      1|        message.add_header("Origin", "http://tp23.org");
  262|      1|        match ws_validate_hdrs(&message) {
  263|      1|            Err(WsUpgradeError::ProtocolError) => {
  264|      1|                // expected
  265|      1|            }
  266|      0|            _ => panic!("Upgrade header check failed"),
  267|       |        }
  268|      1|    }
  269|       |
  270|       |    // TODO we could ignore all other errors??
  271|       |}

src/web_socket/ws_response.rs:
    1|      0|#[derive(Debug, PartialEq)]
    2|       |pub enum FrameType {
    3|       |    NonFinText,
    4|       |    FinCont,
    5|       |    NonFinCont,
    6|       |    FinText,
    7|       |}
    8|       |
    9|       |pub const ECG_FRAME: [u8; 3] = [0x81, 0x01, b'\n'];
   10|       |
   11|       |/**
   12|       | * Write a websockets frame header.
   13|       | * @param out, space to write to, should be 10 bytes long.
   14|       | * @param message_len the length of the ws frame this header prefixes.
   15|       | * @param type frame type
   16|       | * @return the amount of bytes in the header, 2, 4 or 10.
   17|       | */
   18|      5|pub fn ws_write_frame_hdr(out: &mut Box<[u8]>, message_len: usize, frame_type: FrameType) -> usize {
   19|      5|    match frame_type {
   20|      5|        FrameType::NonFinText => {
   21|      0|            out[0] = 0x01; // non-FIN text
   22|      0|        }
   23|      0|        FrameType::FinCont => {
   24|      0|            out[0] = 0x80; // FIN + cont
   25|      0|        }
   26|      0|        FrameType::NonFinCont => {
   27|      0|            out[0] = 0x00; // non-FIN cont
   28|      0|        }
   29|      5|        FrameType::FinText => {
   30|      5|            out[0] = 0x81; // FIN text
   31|      5|        }
   32|       |    }
   33|       |
   34|      5|    let mut i: usize = 0;
   35|      5|    if message_len < 126 {
   36|      3|        out[1] = message_len as u8;
   37|      3|        return 2;
   38|      2|    } else if message_len <= 0xffff {
   39|      1|        out[1] = 0x7e;
   40|      3|        while i < 2 {
   41|      2|            out[2 + i] = (message_len >> 8 * (1 - i) & 0xff) as u8;
   42|      2|            i += 1;
   43|      2|        }
   44|      1|        return 4;
   45|       |    } else {
   46|      1|        out[1] = 0x7f;
   47|      9|        while i < 8 {
   48|      8|            out[2 + i] = (message_len >> 8 * (7 - i) & 0xff) as u8;
   49|      8|            i += 1;
   50|      8|        }
   51|      1|        return 10;
   52|       |    }
   53|      5|}
   54|       |
   55|       |#[cfg(test)]
   56|       |mod tests {
   57|       |    use super::*;
   58|       |    use std::borrow::Borrow;
   59|       |
   60|      1|    #[test]
   61|      1|    fn test_frame_hdr() {
   62|      1|        println!("hdr data={:?}", ECG_FRAME);
   63|      1|
   64|      1|        let mut out: Box<[u8]> = Box::new([0; 2048]);
   65|      1|        assert_eq!(2, ws_write_frame_hdr(&mut out, 1, FrameType::FinText));
   66|      1|        assert_eq!(1, to_int(&out[1..2], 1));
   67|      1|        println!("hdr data={:?}", out[0..2].borrow());
   68|      1|
   69|      1|        assert_eq!(4, ws_write_frame_hdr(&mut out, 0xfffe, FrameType::FinText));
   70|      1|        assert_eq!(0xfffe, to_int(&out[2..4], 2));
   71|      1|        println!("hdr data={:?}", out[0..4].borrow());
   72|      1|
   73|      1|        assert_eq!(
   74|      1|            10,
   75|      1|            ws_write_frame_hdr(&mut out, 0xfefefefe, FrameType::FinText)
   76|      1|        );
   77|      1|        assert_eq!(0xfefefefe, to_int(&out[6..10], 4));
   78|      1|        println!("hdr data={:?}", out[0..10].borrow());
   79|      1|    }
   80|       |
   81|      3|    fn to_int(out: &[u8], len: usize) -> usize {
   82|      3|        const TWO56: usize = 256;
   83|      3|        return match len {
   84|      1|            1 => out[0] as usize,
   85|      1|            2 => (out[0] as usize * TWO56) + out[1] as usize,
   86|       |            // N.B should be 8 digits I'm being lazy
   87|       |            4 => {
   88|      1|                (out[0] as usize * TWO56 * TWO56 * TWO56)
   89|      1|                    + (out[1] as usize * TWO56 * TWO56)
   90|      1|                    + (out[2] as usize * TWO56)
   91|      1|                    + out[3] as usize
   92|       |            }
   93|      0|            _ => panic!(),
   94|       |        };
   95|      3|    }
   96|       |}

src/workflow/console/filter.rs:
    1|       |use std::sync::Arc;
    2|       |
    3|       |use crate::prelude::*;
    4|       |
    5|       |use crate::init::NGIN;
    6|       |
    7|       |/// handles HTTP requests for `xtomp-console`.
    8|       |
    9|      0|#[derive(Debug)]
   10|       |pub struct ConsoleFilter {}
   11|       |
   12|       |impl ConsoleFilter {
   13|      1|    pub fn new() -> ConsoleFilter {
   14|      1|        ConsoleFilter {}
   15|      1|    }
   16|       |}
   17|       |
   18|       |impl HttpFilter for ConsoleFilter {
   19|      0|    fn init(&mut self) {}
   20|       |
   21|      0|    fn do_filter(
   22|      0|        &self,
   23|      0|        context: &mut Context,
   24|      0|        _message: &mut StompMessage,
   25|      0|    ) -> Result<bool, HttpError> {
   26|      0|        let uri = context.attributes.get("romp.uri").unwrap().trim();
   27|      0|        if uri.eq("/romp/ngin-config/") {
   28|      0|            let mut session = context.session.write().unwrap();
   29|      0|            session.send_message(Arc::new(NGIN.response_ngin_config()));
   30|      0|            session.shutdown();
   31|      0|            return Ok(HANDLED);
   32|      0|        }
   33|      0|        if uri.eq("/romp/ngin-info/") {
   34|      0|            let mut session = context.session.write().unwrap();
   35|      0|            session.send_message(Arc::new(NGIN.response_ngin_info()));
   36|      0|            session.shutdown();
   37|      0|            return Ok(HANDLED);
   38|      0|        }
   39|      0|        if uri.starts_with("/romp/destination-info/?destination=") {
   40|       |            // TODO be a bit more forgiving with URL syntax
   41|      0|            let destination = uri.split("=").skip(1).next().unwrap();
   42|      0|
   43|      0|            let mut session = context.session.write().unwrap();
   44|      0|            session.send_message(Arc::new(NGIN.response_destination_info(destination)));
   45|      0|            session.shutdown();
   46|      0|            return Ok(HANDLED);
   47|      0|        }
   48|      0|
   49|      0|        Ok(CONTINUE)
   50|      0|    }
   51|       |}

src/workflow/console/ngin.rs:
    1|       |//! Holds the romp server details in memory in such a way that the data can be read by the
    2|       |//! API used by `xtomp-console`. `xtomp-console` has a frontend HTML/JavaScript component and a backend
    3|       |//! in nodejs.  This code mimics the backend part inside the romp server itself, since we support a
    4|       |//! sufficient sub-set of HTTP.  The term "ngin" comes from nginx the base for xtomp. Naming conventions
    5|       |//! in this code map to similar code in `xtomp-console`.
    6|       |
    7|       |use std::fs::File;
    8|       |use std::io::{BufRead, BufReader};
    9|       |use std::sync::atomic::{AtomicI64, AtomicUsize, Ordering};
   10|       |use std::{fs, process};
   11|       |
   12|       |use crate::init::CONFIG;
   13|       |use crate::init::SERVER;
   14|       |use crate::message::response::{get_http_404_msg, SERVER_TAG};
   15|       |use crate::message::stomp_message::MessageType::Http;
   16|       |use crate::message::stomp_message::{Header, Ownership, StompCommand, StompMessage};
   17|       |use crate::workflow::console::filter::ConsoleFilter;
   18|       |use crate::workflow::http_router::insert_http_filter;
   19|       |use chrono::Utc;
   20|       |use std::sync::RwLock;
   21|       |
   22|       |// inner structs
   23|       |
   24|       |struct DestinationInfo {
   25|       |    size: usize,
   26|       |    queue_len: usize,
   27|       |    delta: usize,
   28|       |    total: usize,
   29|       |}
   30|       |
   31|       |struct NginInfo {
   32|       |    /// contains pre-formatted JSON for the /ngin-config/ response
   33|       |    ngin_config_json: String,
   34|       |    // TODO how does AtomicPtr work, should be able to swap a pointer to DestinationInfo, but I cant get it to work, it will not accept a Box
   35|       |    destinations: Box<Vec<RwLock<DestinationInfo>>>,
   36|       |}
   37|       |
   38|       |/// Provides access to the data required for xtomp-console.
   39|       |pub struct Ngin {
   40|       |    pub enabled: bool,
   41|       |    ngin_info: NginInfo,
   42|       |    connections_count: AtomicUsize,
   43|       |    connections_total: AtomicUsize,
   44|       |    messages_total: AtomicUsize,
   45|       |    lastmod: AtomicI64,
   46|       |}
   47|       |
   48|       |impl Ngin {
   49|       |    /// Create a new Ngin instance, requires CONFIG to be available.
   50|      1|    pub(crate) fn new() -> Ngin {
   51|      1|        let mut ngin_info = NginInfo {
   52|      1|            ngin_config_json: Ngin::serialize_ngin_config(),
   53|      1|            destinations: Box::new(vec![]),
   54|      1|        };
   55|     10|        for _destination in CONFIG.destinations.values() {
   56|     10|            ngin_info.destinations.push(RwLock::new(DestinationInfo {
   57|     10|                size: 0,
   58|     10|                queue_len: 0,
   59|     10|                delta: 0,
   60|     10|                total: 0,
   61|     10|            }));
   62|     10|        }
   63|      1|        Ngin {
   64|      1|            enabled: CONFIG.enable_console,
   65|      1|            ngin_info,
   66|      1|            connections_count: AtomicUsize::new(0),
   67|      1|            connections_total: AtomicUsize::new(0),
   68|      1|            messages_total: AtomicUsize::new(0),
   69|      1|            lastmod: AtomicI64::new(0),
   70|      1|        }
   71|      1|    }
   72|       |
   73|      1|    pub(crate) fn init(&self) {
   74|      1|        insert_http_filter(Box::new(ConsoleFilter::new()));
   75|      1|    }
   76|       |
   77|       |    // mutators //
   78|       |
   79|       |    /// copy this data once every 60 seconds
   80|      0|    pub(crate) fn set_stats_cc(&self) {
   81|      0|        self.connections_count.store(
   82|      0|            SERVER.connections_count.load(Ordering::Relaxed),
   83|      0|            Ordering::Relaxed,
   84|      0|        );
   85|      0|        self.connections_total.store(
   86|      0|            SERVER.connections_total.load(Ordering::Relaxed),
   87|      0|            Ordering::Relaxed,
   88|      0|        );
   89|      0|        self.messages_total.store(
   90|      0|            SERVER.messages_total.load(Ordering::Relaxed),
   91|      0|            Ordering::Relaxed,
   92|      0|        );
   93|      0|        self.lastmod
   94|      0|            .store(Utc::now().timestamp(), Ordering::Relaxed);
   95|      0|    }
   96|       |
   97|       |    pub(crate) fn set_stats_destination(
   98|       |        &self,
   99|       |        idx: usize,
  100|       |        size: usize,
  101|       |        q_size: usize,
  102|       |        delta: usize,
  103|       |        total: usize,
  104|       |    ) {
  105|      0|        if let Ok(mut destination_info) = self.ngin_info.destinations[idx].write() {
  106|      0|            destination_info.size = size;
  107|      0|            destination_info.queue_len = q_size;
  108|      0|            destination_info.delta = delta;
  109|      0|            destination_info.total = total;
  110|      0|        }
  111|      0|    }
  112|       |
  113|       |    // HTTP responses //
  114|       |
  115|       |    /// response for /romp/ngin-config/
  116|      0|    pub fn response_ngin_config(&self) -> StompMessage {
  117|      0|        if !self.enabled {
  118|      0|            return get_http_404_msg();
  119|      0|        }
  120|      0|        // horrible amount of cloning string data here. such is rust.
  121|      0|        let mut message = StompMessage::new(Ownership::Session);
  122|      0|        message.message_type = Http;
  123|      0|        message.command = StompCommand::Ok;
  124|      0|        message.add_header("server", SERVER_TAG);
  125|      0|        message.add_header("access-control-allow-origin", "*");
  126|      0|        message.add_header("content-type", "application/json");
  127|      0|        message.add_header("cache-control", "no-cache");
  128|      0|        let json = &self.ngin_info.ngin_config_json;
  129|      0|        message.push_header(Header {
  130|      0|            name: "content-length".to_string(),
  131|      0|            value: json.len().to_string(),
  132|      0|        });
  133|      0|        message.add_body(json.as_bytes());
  134|      0|        message
  135|      0|    }
  136|       |
  137|       |    /// response for /romp/ngin-info/
  138|      0|    pub fn response_ngin_info(&self) -> StompMessage {
  139|      0|        if !self.enabled {
  140|      0|            return get_http_404_msg();
  141|      0|        }
  142|      0|        let mut message = StompMessage::new(Ownership::Session);
  143|      0|        message.message_type = Http;
  144|      0|        message.command = StompCommand::Ok;
  145|      0|        message.add_header("server", SERVER_TAG);
  146|      0|        message.add_header("access-control-allow-origin", "*");
  147|      0|        message.add_header("content-type", "application/json");
  148|      0|        message.add_header("cache-control", "no-cache");
  149|      0|        let json = self.serialize_ngin_info();
  150|      0|        message.push_header(Header {
  151|      0|            name: "content-length".to_string(),
  152|      0|            value: json.len().to_string(),
  153|      0|        });
  154|      0|        message.add_body(json.as_bytes());
  155|      0|        message
  156|      0|    }
  157|       |
  158|       |    /// response for /romp/destination-info/?destination=memtop-a
  159|      0|    pub fn response_destination_info(&self, destination: &str) -> StompMessage {
  160|      0|        if !self.enabled {
  161|      0|            return get_http_404_msg();
  162|      0|        }
  163|      0|        let mut idx: usize = usize::max_value();
  164|      0|        for (_i, (name, _config)) in SERVER.iter().enumerate() {
  165|      0|            if destination.eq(name) {
  166|      0|                idx = 1;
  167|      0|                break;
  168|      0|            }
  169|       |        }
  170|      0|        if idx == usize::max_value() {
  171|      0|            return get_http_404_msg();
  172|      0|        }
  173|      0|
  174|      0|        let mut message = StompMessage::new(Ownership::Session);
  175|      0|        message.message_type = Http;
  176|      0|        message.command = StompCommand::Ok;
  177|      0|        message.add_header("server", SERVER_TAG);
  178|      0|        message.add_header("access-control-allow-origin", "*");
  179|      0|        message.add_header("content-type", "application/json");
  180|      0|        message.add_header("cache-control", "no-cache");
  181|      0|        let json = self.serialize_destination_info(idx);
  182|      0|        message.push_header(Header {
  183|      0|            name: "content-length".to_string(),
  184|      0|            value: json.len().to_string(),
  185|      0|        });
  186|      0|        message.add_body(json.as_bytes());
  187|      0|        message
  188|      0|    }
  189|       |
  190|       |    // ngin-info //
  191|       |
  192|       |    // the following code presumes kernel `/proc` interfaces are stable.
  193|       |    // could process the data a little more but this code should match ngin-info.js in xtomp-console
  194|       |
  195|       |    /// return JSON representation of `/proc/pid/...` data relevant to the `romp` process.
  196|      0|    fn serialize_ngin_info(&self) -> String {
  197|      0|        format!(
  198|      0|            "{{ {}, {}, {}, {}, {} }}",
  199|      0|            Ngin::serialize_proc_pid(),
  200|      0|            Ngin::serialize_proc_cmdline(),
  201|      0|            Ngin::serialize_proc_sockstat(),
  202|      0|            Ngin::serialize_proc_status(),
  203|      0|            self.serialize_cc()
  204|      0|        )
  205|      0|    }
  206|       |
  207|       |    /// return a JSON string
  208|      1|    fn serialize_proc_pid() -> String {
  209|      1|        format!("\"pid\": {}", process::id())
  210|      1|    }
  211|       |
  212|       |    /// reads `/proc/pid/cmdline` presuming this is a valid JSON string, and return a JSON string
  213|      1|    fn serialize_proc_cmdline() -> String {
  214|      1|        let mut path = String::from("/proc/");
  215|      1|        path.push_str(process::id().to_string().as_str());
  216|      1|        path.push_str("/cmdline");
  217|      1|        let cmdline = fs::read_to_string(path).expect("Unable /proc/[pid]/cmdline");
  218|      1|        format!(
  219|      1|            "\"cmdline\": \"{}\"",
  220|      1|            cmdline.trim_matches(char::from(0)).trim()
  221|      1|        )
  222|      1|    }
  223|       |
  224|       |    /// reads `/proc/pid/net/sockstat` presuming this contains string that do not
  225|       |    /// need escaping and returns a JSON string
  226|      1|    fn serialize_proc_sockstat() -> String {
  227|      1|        let mut path = String::from("/proc/");
  228|      1|        path.push_str(process::id().to_string().as_str());
  229|      1|        path.push_str("/net/sockstat");
  230|      1|        let f = File::open(path).expect("Unable to read /proc/[pid]/net/sockstat");
  231|      1|        let f = BufReader::new(f);
  232|      1|
  233|      1|        let mut json = String::from("\"sock\": {");
  234|      6|        for line in f.lines() {
  235|      6|            let line = line.expect("Unable to read /proc/[pid]/net/sockstat");
  236|      6|            if line.starts_with("sockets") {
  237|      1|                let parts: Vec<&str> = line.split(":").collect();
  238|      1|                if parts.len() == 2 {
  239|      1|                    json.push_str("\"sockets\":\"");
  240|      1|                    json.push_str(parts[1].trim());
  241|      1|                    json.push_str("\",");
  242|      1|                }
  243|      5|            }
  244|      6|            if line.starts_with("TCP") {
  245|      1|                let parts: Vec<&str> = line.split(":").collect();
  246|      1|                if parts.len() == 2 {
  247|      1|                    json.push_str("\"tcp\":\"");
  248|      1|                    json.push_str(parts[1].trim());
  249|      1|                    json.push_str("\",");
  250|      1|                }
  251|      5|            }
  252|       |        }
  253|      1|        if json.ends_with(",") {
  254|      1|            json.remove(json.len() - 1);
  255|      1|        }
  256|      1|        json.push_str("}");
  257|      1|        json
  258|      1|    }
  259|       |
  260|      1|    fn serialize_proc_status() -> String {
  261|      1|        let mut path = String::from("/proc/");
  262|      1|        path.push_str(process::id().to_string().as_str());
  263|      1|        path.push_str("/status");
  264|      1|        let f = File::open(path).expect("Unable to read /proc/[pid]/status");
  265|      1|        let f = BufReader::new(f);
  266|      1|
  267|      1|        let mut json = String::from("\"proc\": {");
  268|     57|        for line in f.lines() {
  269|     57|            let line = line.expect("Unable to read /proc/[pid]/status");
  270|     57|            if line.starts_with("Vm") {
  271|     12|                let parts: Vec<&str> = line.split(":").collect();
  272|     12|                if parts.len() == 2 {
  273|     12|                    json.push_str("\"");
  274|     12|                    json.push_str(parts[0]);
  275|     12|                    json.push_str("\":\"");
  276|     12|                    json.push_str(parts[1].trim());
  277|     12|                    json.push_str("\",");
  278|     12|                }
  279|     45|            }
  280|       |        }
  281|      1|        if json.ends_with(",") {
  282|      1|            json.remove(json.len() - 1);
  283|      1|        }
  284|      1|        json.push_str("}");
  285|      1|        json
  286|      1|    }
  287|       |
  288|      0|    fn serialize_cc(&self) -> String {
  289|      0|        let mut json = String::from("\"cc\": {");
  290|      0|        json.push_str(
  291|      0|            format!(
  292|      0|                "\"sz\": \"{}\",",
  293|      0|                self.connections_count.load(Ordering::Relaxed)
  294|      0|            )
  295|      0|            .as_str(),
  296|      0|        );
  297|      0|        json.push_str(
  298|      0|            format!(
  299|      0|                "\"up\": \"{}\",",
  300|      0|                (SERVER.uptime.load(Ordering::Relaxed)) / 1000
  301|      0|            )
  302|      0|            .as_str(),
  303|      0|        );
  304|      0|        json.push_str(
  305|      0|            format!(
  306|      0|                "\"Σ\": \"{}\",",
  307|      0|                self.connections_total.load(Ordering::Relaxed)
  308|      0|            )
  309|      0|            .as_str(),
  310|      0|        );
  311|      0|        json.push_str(
  312|      0|            format!(
  313|      0|                "\"Σµ\": \"{}\",",
  314|      0|                self.messages_total.load(Ordering::Relaxed)
  315|      0|            )
  316|      0|            .as_str(),
  317|      0|        );
  318|      0|        json.push_str("\"m\": \"0\",");
  319|      0|        json.push_str(
  320|      0|            format!("\"lastmod\": \"{}\"", self.lastmod.load(Ordering::Relaxed)).as_str(),
  321|      0|        );
  322|      0|        json.push_str("}");
  323|      0|        json
  324|      0|    }
  325|       |
  326|       |    // ngin-config //
  327|       |
  328|      1|    fn serialize_ngin_config() -> String {
  329|      1|        format!(
  330|      1|            "{{ {}, {}, {}, {} }}",
  331|      1|            Ngin::serialize_listen(),
  332|      1|            Ngin::serialize_timeout(),
  333|      1|            Ngin::serialize_websockets(),
  334|      1|            Ngin::serialize_destinations()
  335|      1|        )
  336|      1|    }
  337|       |
  338|      1|    fn serialize_listen() -> String {
  339|      1|        let mut json = String::from("\"listen\": [");
  340|      2|        for port in &CONFIG.ports {
  341|      2|            json.push_str(port.to_string().as_str());
  342|      2|            json.push_str(",");
  343|      2|        }
  344|      1|        if json.ends_with(",") {
  345|      1|            json.remove(json.len() - 1);
  346|      1|        }
  347|      1|        json.push_str("]");
  348|      1|        json
  349|      1|    }
  350|       |
  351|      1|    fn serialize_timeout() -> String {
  352|      1|        String::from("\"timeout\": 0")
  353|      1|    }
  354|       |
  355|      1|    fn serialize_websockets() -> String {
  356|      1|        format!("\"websockets\": {}", CONFIG.websockets)
  357|      1|    }
  358|       |
  359|      1|    fn serialize_destinations() -> String {
  360|      1|        let mut json = String::from("\"destinations\": [");
  361|       |
  362|     10|        for destination in CONFIG.destinations.values() {
  363|     10|            json.push_str("{");
  364|     10|            json.push_str(format!("\"name\": \"{}\",", destination.name).as_str());
  365|     10|            json.push_str(format!("\"filter\": {},", destination.filter).as_str());
  366|     10|            // currently not supported in romp
  367|     10|            // json.push_str(format!("\"filter_header\": \"{}\",", destination.filter_header).as_str());
  368|     10|            json.push_str(
  369|     10|                format!("\"max_connections\": {},", destination.max_connections).as_str(),
  370|     10|            );
  371|     10|            json.push_str(format!("\"max_messages\": {},", destination.max_messages).as_str());
  372|     10|            json.push_str(format!("\"stats\": {},", destination.stats).as_str());
  373|     10|            json.push_str(
  374|     10|                format!("\"web_write_block\": {},", destination.web_write_block).as_str(),
  375|     10|            );
  376|     10|            json.push_str(format!("\"web_read_block\": {},", destination.web_read_block).as_str());
  377|     10|            json.push_str(
  378|     10|                format!("\"max_message_size\": {}", destination.max_message_size).as_str(),
  379|     10|            );
  380|     10|            json.push_str("},");
  381|     10|        }
  382|      1|        if json.ends_with(",") {
  383|      1|            json.remove(json.len() - 1);
  384|      1|        }
  385|      1|        json.push_str("]");
  386|      1|        json
  387|      1|    }
  388|       |
  389|       |    // destination-info //
  390|       |
  391|       |    fn serialize_destination_info(&self, idx: usize) -> String {
  392|      0|        if let Ok(destination_info) = self.ngin_info.destinations[idx].read() {
  393|      0|            return format!(
  394|      0|                "{{\"sz\":{}, \"q\":{}, \"Δ\":{}, \"Σ\":{}}}",
  395|      0|                destination_info.size,
  396|      0|                destination_info.queue_len,
  397|      0|                destination_info.delta,
  398|      0|                destination_info.total
  399|      0|            );
  400|      0|        }
  401|      0|        String::from("{}")
  402|      0|    }
  403|       |}
  404|       |
  405|       |#[cfg(test)]
  406|       |mod tests {
  407|       |
  408|       |    use super::*;
  409|       |
  410|      1|    #[test]
  411|      1|    fn test() {
  412|      1|        println!("{}", Ngin::serialize_proc_pid());
  413|      1|        println!("{}", Ngin::serialize_proc_cmdline());
  414|      1|        println!("{}", Ngin::serialize_proc_sockstat());
  415|      1|        println!("{}", Ngin::serialize_proc_status());
  416|      1|    }
  417|       |}

src/workflow/destination/destination.rs:
    1|       |//!Destination represents a single queue or topic.
    2|       |
    3|       |use std::collections::HashMap;
    4|       |use std::slice::Iter;
    5|       |use std::sync::atomic::{AtomicUsize, Ordering};
    6|       |use std::sync::{Arc, RwLock};
    7|       |
    8|       |use chrono::Utc;
    9|       |use futures::task::Task;
   10|       |use log::*;
   11|       |
   12|       |use crate::config::config;
   13|       |use crate::errors::ServerError;
   14|       |use crate::init::SERVER;
   15|       |use crate::message::stomp_message::{Header, Ownership, StompCommand, StompMessage};
   16|       |use crate::session::subscription::Subscription;
   17|       |use std::time::Duration;
   18|       |
   19|       |/// A Queue / Topic for messages
   20|       |/// Messages are send in strict order to all subscribers, when min_delivery messages have been sent
   21|       |/// the message is removed from the front of the queue.  Until then the queue does not progress.
   22|       |// TODO support for destinations that send any messages they get immediately
   23|       |
   24|       |pub struct Destination {
   25|       |    // data
   26|       |    name: String,
   27|       |    id: usize,
   28|       |    q: Box<Vec<Arc<StompMessage>>>,
   29|       |    subs: HashMap<String, Arc<RwLock<Subscription>>>,
   30|       |
   31|       |    // config
   32|       |    /// maximum number of subscribers
   33|       |    max_connections: usize,
   34|       |    /// maximum number of message to store on the queue
   35|       |    max_messages: usize,
   36|       |    /// number of times a message must be sent to remove it from the queue, set min_delivery==1 for a queue,  set to 2 for a queue with redundancy, set to zero for a topic that discards messages if no one is listening
   37|       |    min_delivery: usize,
   38|       |    /// default expiry time
   39|       |    expiry: u64,
   40|       |    /// check each message has not expired before sending it
   41|       |    pedantic_expiry: bool,
   42|       |    filter: bool,
   43|       |    filter_self: bool,
   44|       |    sub_by_user: bool,
   45|       |    record_stats: bool,
   46|       |
   47|       |    // security
   48|       |    /// force all subs be auto ack (otherwise a client can set ack:client and hang the topic by no Acking the head)
   49|       |    auto_ack: bool,
   50|       |    /// limit size of uploaded messages
   51|       |    max_message_size: usize,
   52|       |    /// prevent reads from the network
   53|       |    read_block: bool,
   54|       |    /// prevent writes from the network
   55|       |    write_block: bool,
   56|       |    /// prevent reads from t'interwebs
   57|       |    web_read_block: bool,
   58|       |    /// prevent writes from t'interwebs
   59|       |    web_write_block: bool,
   60|       |
   61|       |    // stats
   62|       |    message_delta: AtomicUsize,
   63|       |    message_total: AtomicUsize,
   64|       |
   65|       |    // state
   66|       |    task: Option<Task>,
   67|       |    /// when true accept no more messages
   68|       |    drain: bool,
   69|       |    /// when true its time to die
   70|       |    shutdown: bool,
   71|       |    /// set to true when the head of the queue has not been delivered enough times (typically min_delivery==1 for a queue)
   72|       |    queued: bool,
   73|       |}
   74|       |
   75|       |/// attributes that are configurable at runtime
   76|       |pub struct RuntimeConfig {
   77|       |    pub max_connections: Option<usize>,
   78|       |    pub max_messages: Option<usize>,
   79|       |    pub expiry: Option<u64>,
   80|       |    pub max_message_size: Option<usize>,
   81|       |    pub write_block: Option<bool>,
   82|       |    pub read_block: Option<bool>,
   83|       |    pub web_write_block: Option<bool>,
   84|       |    pub web_read_block: Option<bool>,
   85|       |}
   86|       |
   87|       |impl Default for RuntimeConfig {
   88|      0|    fn default() -> Self {
   89|      0|        RuntimeConfig {
   90|      0|            max_connections: None,
   91|      0|            max_messages: None,
   92|      0|            expiry: None,
   93|      0|            max_message_size: None,
   94|      0|            write_block: None,
   95|      0|            read_block: None,
   96|      0|            web_write_block: None,
   97|      0|            web_read_block: None,
   98|      0|        }
   99|      0|    }
  100|       |}
  101|       |
  102|       |impl Destination {
  103|     15|    pub(crate) fn new(name: String, id: usize, cfg: &config::Destination) -> Destination {
  104|     15|        Destination {
  105|     15|            name,
  106|     15|            q: Box::new(vec![]),
  107|     15|            id,
  108|     15|            subs: Default::default(),
  109|     15|
  110|     15|            // config
  111|     15|            max_connections: cfg.max_connections,
  112|     15|            max_messages: cfg.max_messages,
  113|     15|            min_delivery: cfg.min_delivery,
  114|     15|            expiry: cfg.expiry,
  115|     15|            pedantic_expiry: cfg.pedantic_expiry,
  116|     15|            filter: cfg.filter,
  117|     15|            filter_self: cfg.filter_self,
  118|     15|            sub_by_user: cfg.sub_by_user,
  119|     15|            record_stats: cfg.stats,
  120|     15|
  121|     15|            // security
  122|     15|            auto_ack: cfg.auto_ack,
  123|     15|            max_message_size: cfg.max_message_size,
  124|     15|            read_block: cfg.read_block,
  125|     15|            write_block: cfg.write_block,
  126|     15|            web_read_block: cfg.web_read_block,
  127|     15|            web_write_block: cfg.web_write_block,
  128|     15|
  129|     15|            // stats
  130|     15|            message_delta: AtomicUsize::new(0),
  131|     15|            message_total: AtomicUsize::new(0),
  132|     15|
  133|     15|            // state
  134|     15|            task: None,
  135|     15|            drain: false,
  136|     15|            shutdown: false,
  137|     15|            queued: false,
  138|     15|        }
  139|     15|    }
  140|       |
  141|       |    // accessors
  142|       |
  143|      0|    pub fn name(&self) -> &String {
  144|      0|        &self.name
  145|      0|    }
  146|       |
  147|     12|    pub fn id(&self) -> usize {
  148|     12|        self.id
  149|     12|    }
  150|       |
  151|      0|    pub fn expiry(&self) -> u64 {
  152|      0|        self.expiry
  153|      0|    }
  154|       |
  155|      1|    pub fn auto_ack(&self) -> bool {
  156|      1|        self.auto_ack
  157|      1|    }
  158|       |
  159|      0|    pub fn write_block(&self) -> bool {
  160|      0|        self.write_block
  161|      0|    }
  162|       |
  163|      0|    pub fn read_block(&self) -> bool {
  164|      0|        self.read_block
  165|      0|    }
  166|       |
  167|      0|    pub fn web_write_block(&self) -> bool {
  168|      0|        self.web_write_block
  169|      0|    }
  170|       |
  171|      0|    pub fn web_read_block(&self) -> bool {
  172|      0|        self.web_read_block
  173|      0|    }
  174|       |
  175|      0|    pub fn record_stats(&self) -> bool {
  176|      0|        self.record_stats
  177|      0|    }
  178|       |
  179|      0|    pub fn filter_self(&self) -> bool {
  180|      0|        self.filter_self
  181|      0|    }
  182|       |
  183|      0|    pub fn max_connections(&self) -> usize {
  184|      0|        self.max_connections
  185|      0|    }
  186|       |
  187|      0|    pub fn message_total(&self) -> usize {
  188|      0|        self.message_total.load(Ordering::Relaxed)
  189|      0|    }
  190|       |
  191|      0|    pub fn message_delta(&self) -> usize {
  192|      0|        self.message_delta.swap(0, Ordering::Relaxed)
  193|      0|    }
  194|       |
  195|      0|    pub fn len(&self) -> usize {
  196|      0|        self.q.len()
  197|      0|    }
  198|       |
  199|      0|    pub fn iter(&mut self) -> Iter<Arc<StompMessage>> {
  200|      0|        self.q.iter()
  201|      0|    }
  202|       |
  203|       |    // mutators
  204|       |
  205|      0|    pub(crate) fn set_task(&mut self, task: Task) {
  206|      0|        debug!("captured task");
  207|      0|        self.task = Some(task);
  208|      0|    }
  209|       |
  210|      0|    pub fn reconfigure(&mut self, cfg: RuntimeConfig) {
  211|      0|        if cfg.max_connections.is_some() {
  212|      0|            self.max_connections = cfg.max_connections.unwrap();
  213|      0|        }
  214|      0|        if cfg.max_messages.is_some() {
  215|      0|            self.max_messages = cfg.max_messages.unwrap();
  216|      0|        }
  217|      0|        if cfg.expiry.is_some() {
  218|      0|            self.expiry = cfg.expiry.unwrap();
  219|      0|        }
  220|      0|        if cfg.max_message_size.is_some() {
  221|      0|            self.max_message_size = cfg.max_message_size.unwrap();
  222|      0|        }
  223|      0|        if cfg.read_block.is_some() {
  224|      0|            self.read_block = cfg.read_block.unwrap();
  225|      0|        }
  226|      0|        if cfg.write_block.is_some() {
  227|      0|            self.write_block = cfg.write_block.unwrap();
  228|      0|        }
  229|      0|        if cfg.web_read_block.is_some() {
  230|      0|            self.web_read_block = cfg.web_read_block.unwrap();
  231|      0|        }
  232|      0|        if cfg.web_write_block.is_some() {
  233|      0|            self.web_write_block = cfg.web_write_block.unwrap();
  234|      0|        }
  235|      0|    }
  236|       |
  237|       |    /// Put a copy of the message on the q
  238|       |    // TODO return future for notification of durability
  239|      7|    pub fn push(&mut self, message: &StompMessage) -> Result<usize, ServerError> {
  240|      7|        return match self.push_security(&message) {
  241|      0|            Err(se) => Err(se),
  242|       |            Ok(_) => {
  243|      7|                let id = SERVER.new_message_id();
  244|      7|                self.message_delta.fetch_add(1, Ordering::SeqCst);
  245|      7|                self.message_total.fetch_add(1, Ordering::SeqCst);
  246|      7|
  247|      7|                let mut copy = message.clone_to_message(Ownership::Destination, id);
  248|      7|                copy.expiry = Duration::from_millis(self.expiry);
  249|      7|
  250|      7|                self.q.push(Arc::new(copy));
  251|      7|
  252|      7|                self.notify();
  253|      7|
  254|      7|                Ok(id)
  255|       |            }
  256|       |        };
  257|      7|    }
  258|       |
  259|       |    /// put an owned message on the q
  260|      0|    pub fn push_owned(&mut self, mut message: StompMessage) -> Result<usize, ServerError> {
  261|      0|        return match self.push_security(&message) {
  262|      0|            Err(se) => Err(se),
  263|       |            Ok(_) => {
  264|      0|                let id = message.id;
  265|      0|                if id == 0 {
  266|      0|                    let id = SERVER.new_message_id();
  267|      0|                    self.message_total.fetch_add(1, Ordering::SeqCst);
  268|      0|                    message.id = id;
  269|      0|                }
  270|       |
  271|      0|                message.expiry = Duration::from_millis(self.expiry);
  272|      0|
  273|      0|                self.q.push(Arc::new(message));
  274|      0|
  275|      0|                self.notify();
  276|      0|
  277|      0|                Ok(id)
  278|       |            }
  279|       |        };
  280|      0|    }
  281|       |
  282|       |    /// validation that its safe to push this message to the destination
  283|      7|    fn push_security(&mut self, message: &StompMessage) -> Result<(), ServerError> {
  284|      7|        if self.drain {
  285|      0|            return Err(ServerError::ShuttingDown);
  286|      7|        }
  287|      7|        if self.q.len() == self.max_messages {
  288|      0|            return Err(ServerError::DestinationFlup);
  289|      7|        }
  290|      7|        if message.body_len() > self.max_message_size {
  291|      0|            return Err(ServerError::MessageFlup);
  292|      7|        }
  293|      7|        Ok(())
  294|      7|    }
  295|       |
  296|       |    /// push a message to a specific logged in user, destination must be hashed by username
  297|      0|    pub fn push_direct(&mut self, message: StompMessage) -> bool {
  298|      0|        if self.sub_by_user {
  299|       |            // how to borrow an Option<String>
  300|      0|            let to = &message.to.as_ref();
  301|       |
  302|      0|            if let Some(sub) = self.subs.get(to.unwrap()) {
  303|      0|                return sub
  304|      0|                    .write()
  305|      0|                    .unwrap()
  306|      0|                    .publish(self.id, Arc::new(message), false);
  307|      0|            }
  308|      0|        }
  309|      0|        false
  310|      0|    }
  311|       |
  312|       |    /// forward a downstream MESSAGE to a SEND on this destination
  313|      0|    pub fn push_resend(&mut self, message: &StompMessage) -> Result<usize, ServerError> {
  314|      0|        return match self.push_security(&message) {
  315|      0|            Err(se) => Err(se),
  316|       |            Ok(_) => {
  317|       |                // for now we increment the id because we dont know how upstream allocates ids.
  318|      0|                let id = SERVER.new_message_id();
  319|      0|                let original_id = message.id;
  320|      0|                self.message_delta.fetch_add(1, Ordering::SeqCst);
  321|      0|                self.message_total.fetch_add(1, Ordering::SeqCst);
  322|      0|
  323|      0|                let mut copy = message.clone_to_message(Ownership::Destination, id);
  324|      0|                copy.expiry = Duration::from_millis(self.expiry);
  325|      0|
  326|      0|                if copy.command == StompCommand::Message {
  327|      0|                    copy.command = StompCommand::Send;
  328|      0|                }
  329|      0|                copy.push_header(Header {
  330|      0|                    name: "orig-id".to_string(),
  331|      0|                    value: original_id.to_string(),
  332|      0|                });
  333|      0|
  334|      0|                self.q.push(Arc::new(copy));
  335|      0|
  336|      0|                self.notify();
  337|      0|
  338|      0|                Ok(id)
  339|       |            }
  340|       |        };
  341|      0|    }
  342|       |
  343|       |    pub fn find_sub(&self, to: &String) -> Option<&Arc<RwLock<Subscription>>> {
  344|      0|        return if let Some(sub) = self.subs.get(to) {
  345|      0|            Some(sub)
  346|       |        } else {
  347|      0|            None
  348|       |        };
  349|      0|    }
  350|       |
  351|       |    /// called periodically for message expiry
  352|      0|    pub fn timeout_messages(&mut self) -> Result<(), tokio::timer::Error> {
  353|      0|        self.expire();
  354|      0|        if self.drain || self.shutdown {
  355|      0|            return Err(tokio::timer::Error::shutdown());
  356|      0|        }
  357|      0|        Ok(())
  358|      0|    }
  359|       |
  360|       |    // TODO pushes HashMap responsibility to client, alternative it get a read lock on Session
  361|      0|    pub(crate) fn subscribe(&mut self, sub: Arc<RwLock<Subscription>>) -> Result<(), ServerError> {
  362|      0|        if self.drain || self.shutdown {
  363|      0|            return Err(ServerError::ShuttingDown);
  364|      0|        }
  365|      0|
  366|      0|        if self.subs.len() == self.max_connections {
  367|      0|            return Err(ServerError::SubscriptionFlup);
  368|      0|        }
  369|      0|
  370|      0|        // clone the string because borrow checker cant handle realities of subscription lifetime
  371|      0|        let hash_id;
  372|      0|        {
  373|      0|            if self.sub_by_user {
  374|      0|                hash_id = sub.read().unwrap().user_id();
  375|      0|            } else {
  376|      0|                hash_id = sub.read().unwrap().hash_id().clone();
  377|      0|            }
  378|       |        }
  379|      0|        self.subs.insert(hash_id, sub);
  380|      0|
  381|      0|        if self.queued {
  382|      0|            debug!("subscribe while queued, notify");
  383|      0|            self.notify();
  384|       |        } else {
  385|      0|            debug!("subscribed to {}", self.name);
  386|       |        }
  387|       |
  388|      0|        Ok(())
  389|      0|    }
  390|       |
  391|       |    // Unsubscribe return true on success
  392|      0|    pub(crate) fn unsubscribe(&mut self, id: &String) -> bool {
  393|      0|        self.subs.remove(id).is_some()
  394|      0|    }
  395|       |
  396|      0|    pub fn shutdown(&mut self) {
  397|      0|        self.shutdown = true;
  398|      0|        self.q.clear();
  399|      0|        match &self.task {
  400|      0|            Some(task) => task.notify(),
  401|      0|            _ => {}
  402|       |        }
  403|      0|    }
  404|       |
  405|      0|    pub fn drain(&mut self) {
  406|      0|        self.drain = true;
  407|      0|        self.expire();
  408|      0|        match &self.task {
  409|      0|            Some(task) => task.notify(),
  410|      0|            _ => {}
  411|       |        }
  412|      0|    }
  413|       |
  414|      0|    pub fn clean(&mut self) -> usize {
  415|      0|        let len = self.q.len();
  416|      0|        self.q.clear();
  417|      0|        return len;
  418|      0|    }
  419|       |
  420|      8|    fn notify(&self) {
  421|      8|        match &self.task {
  422|      8|            None => warn!("publish before Destination is started, message queued"),
  423|      0|            Some(task) => {
  424|      0|                debug!("destination notified");
  425|      0|                task.notify()
  426|       |            }
  427|       |        }
  428|      8|    }
  429|       |
  430|       |    /// expire messages that are past their sell by date
  431|      1|    fn expire(&mut self) {
  432|      1|        let now = Utc::now().timestamp() as u64;
  433|      1|        let len = self.q.len();
  434|      1|        self.q.retain(|message| {
  435|      0|            message.timestamp.timestamp() as u64 + message.expiry.as_secs() as u64 > now
  436|      1|        });
  437|      1|        let expired = len - self.q.len();
  438|      1|        if expired > 0 {
  439|      0|            info!("expired {} messages, remain={}", expired, self.q.len());
  440|      1|        }
  441|      1|    }
  442|       |
  443|       |    /// Acknowledge a message as consumed
  444|       |    /// This only makes sense for queues, since we don't track Acks per subscription
  445|      2|    pub fn ack(&self, id: usize) {
  446|      2|        debug!("ACKing {}", id);
  447|      2|        for message in self.q.iter() {
  448|      2|            if message.id == id {
  449|      2|                if message.increment_delivered(1) + 1 >= self.min_delivery {
  450|      2|                    self.notify();
  451|      2|                }
  452|      2|                break;
  453|      0|            }
  454|       |        }
  455|      2|    }
  456|       |
  457|       |    /// Acknowledge a message as unable to be consumed
  458|       |    // TODO DLQ
  459|      0|    pub fn nack(&self, id: usize) {
  460|      0|        self.ack(id);
  461|      0|    }
  462|       |
  463|       |    /// called once each time a message is added to the q, or acked, or a sub
  464|       |    /// return true if we are still alive
  465|      6|    pub(crate) fn poll(&mut self) -> bool {
  466|      6|        debug!("destination polled {} messages on q", self.q.len());
  467|      6|        if self.drain && self.q.len() == 0 {
  468|      0|            warn!("destination {} dead", self.name);
  469|      0|            return false;
  470|      6|        }
  471|      6|
  472|      6|        if self.shutdown {
  473|      0|            warn!("destination {} dead", self.name);
  474|      0|            return false;
  475|      6|        }
  476|      6|
  477|      6|        if self.q.len() == 0 {
  478|      1|            self.queued = false;
  479|      1|            return true;
  480|      5|        }
  481|      5|
  482|      5|        let now = Utc::now().timestamp() as u64;
  483|      9|        while let Some(message) = self.q.get(0) {
  484|      7|            if self.pedantic_expiry {
  485|      0|                let mut do_drop = false;
  486|      0|                {
  487|      0|                    if message.timestamp.timestamp() as u64 + message.expiry.as_secs() as u64 > now
  488|       |                    {
  489|      0|                        info!("expired 1 message");
  490|      0|                        do_drop = true;
  491|      0|                    }
  492|       |                }
  493|      0|                if do_drop {
  494|      0|                    self.q.pop();
  495|      0|                    continue;
  496|      0|                }
  497|      7|            }
  498|       |
  499|      7|            let mut delivered: usize = 0;
  500|      7|            for (key, subscription) in self.subs.iter() {
  501|       |                // publish clone of Arc, only one copy of message exists
  502|       |                {
  503|      0|                    debug!("publishing instance of message to {}", key);
  504|      0|                    if subscription
  505|      0|                        .write()
  506|      0|                        .unwrap()
  507|      0|                        .publish(self.id, message.clone(), self.filter)
  508|      0|                    {
  509|      0|                        delivered += 1;
  510|      0|                    }
  511|       |                }
  512|       |            }
  513|       |
  514|       |            let total_delivered;
  515|      7|            {
  516|      7|                total_delivered = message.increment_delivered(delivered) + delivered;
  517|      7|            }
  518|      7|
  519|      7|            if self.min_delivery == 0 {
  520|      2|                // not tracking delivery
  521|      2|                self.q.remove(0);
  522|      5|            } else if self.min_delivery > total_delivered {
  523|       |                // queue does not progress if don't have enough subscribers
  524|      3|                self.queued = true;
  525|      3|                debug!("queued, not enough subs");
  526|      3|                return true;
  527|      2|            } else {
  528|      2|                // delivered enough times
  529|      2|                self.q.remove(0);
  530|      2|            }
  531|       |        }
  532|       |
  533|      2|        self.queued = false;
  534|      2|
  535|      2|        return true;
  536|      6|    }
  537|       |}
  538|       |
  539|       |#[cfg(test)]
  540|       |mod tests {
  541|       |    use std::time;
  542|       |    use std::time::Duration;
  543|       |
  544|       |    use crate::config::config;
  545|       |
  546|       |    use super::*;
  547|       |
  548|      1|    #[test]
  549|      1|    fn test() {
  550|      1|        let dest_config = config::Destination {
  551|      1|            min_delivery: 1,
  552|      1|            ..Default::default()
  553|      1|        };
  554|      1|        let mut destination = Destination::new(String::from(""), 1, &dest_config);
  555|      1|
  556|      1|        let id_1 = destination
  557|      1|            .push(&StompMessage::new_send(b"{\"test\": true}", 1))
  558|      1|            .unwrap_or(99);
  559|      1|        let id_2 = destination
  560|      1|            .push(&StompMessage::new_send(b"{\"test\": false}", 2))
  561|      1|            .unwrap_or(99);
  562|      1|
  563|      1|        // with no one listening poll should keep messages on the q
  564|      1|        destination.poll();
  565|      1|        assert!(destination.queued);
  566|      1|        assert_eq!(destination.q.len(), 2);
  567|       |
  568|      1|        assert_eq!(destination.q.remove(0).id, id_1);
  569|      1|        assert_eq!(destination.q.remove(0).id, id_2);
  570|       |        // poll empty dest should set queued flag to false
  571|      1|        destination.poll();
  572|      1|        assert!(!destination.queued);
  573|       |
  574|      1|        let id_3 = destination
  575|      1|            .push(&StompMessage::new_send(b"{\"test\": false}", 3))
  576|      1|            .unwrap_or(99);
  577|      1|
  578|      1|        assert_eq!(destination.q.remove(0).id, id_3);
  579|       |
  580|      1|        let mut message = StompMessage::new_send(b"{\"timeout\": true}", 4);
  581|      1|        message.expiry = Duration::new(0, 0);
  582|      1|
  583|      1|        std::thread::sleep(time::Duration::from_millis(10));
  584|      1|
  585|      1|        destination.expire();
  586|      1|        assert_eq!(destination.q.len(), 0);
  587|      1|    }
  588|       |
  589|      1|    #[test]
  590|      1|    fn test_no_min_delivery() {
  591|      1|        let mut destination = Destination::new(
  592|      1|            String::from(""),
  593|      1|            1,
  594|      1|            &config::Destination {
  595|      1|                min_delivery: 0,
  596|      1|                ..Default::default()
  597|      1|            },
  598|      1|        );
  599|      1|
  600|      1|        match destination.push(&StompMessage::new_send(b"{\"test\": true}", 1)) {
  601|      1|            Err(_) => panic!("push failed"),
  602|      1|            _ => {}
  603|      1|        }
  604|      1|        match destination.push(&StompMessage::new_send(b"{\"test\": false}", 2)) {
  605|      1|            Err(_) => panic!("push failed"),
  606|      1|            _ => {}
  607|      1|        }
  608|      1|
  609|      1|        // with min_delivery set to zero poll should empty the q
  610|      1|        destination.poll();
  611|      1|        assert!(!destination.queued);
  612|      1|        assert_eq!(destination.q.len(), 0);
  613|      1|    }
  614|       |
  615|      1|    #[test]
  616|      1|    fn test_ack() {
  617|      1|        let mut destination = Destination::new(
  618|      1|            String::from("ackme"),
  619|      1|            1,
  620|      1|            &config::Destination {
  621|      1|                min_delivery: 1,
  622|      1|                ..Default::default()
  623|      1|            },
  624|      1|        );
  625|      1|
  626|      1|        assert_eq!(destination.auto_ack(), false);
  627|       |
  628|      1|        match destination.push(&StompMessage::new_send(b"{\"test\": true}", 2)) {
  629|      1|            Err(_) => panic!("push failed"),
  630|      1|            _ => {}
  631|      1|        }
  632|      1|        match destination.push(&StompMessage::new_send(b"{\"test\": false}", 4)) {
  633|      1|            Err(_) => panic!("push failed"),
  634|      1|            _ => {}
  635|      1|        }
  636|      1|
  637|      1|        assert_eq!(destination.q.len(), 2);
  638|      1|        destination.poll();
  639|      1|        // nothing acked we should still have the messages
  640|      1|        assert_eq!(destination.queued, true);
  641|      1|        assert_eq!(destination.q.len(), 2);
  642|       |
  643|      1|        destination.ack(destination.q[0].id);
  644|      1|        destination.poll();
  645|      1|        assert_eq!(destination.q.len(), 1);
  646|       |
  647|      1|        destination.ack(destination.q[0].id);
  648|      1|        destination.poll();
  649|      1|        assert_eq!(destination.q.len(), 0);
  650|      1|    }
  651|       |}

src/workflow/destination/server.rs:
    1|       |//! DestinationServer is the in memory representation of queues and topics.
    2|       |//! Holds all the servers Destinations, which contain messages to be sent out and subscriber information.
    3|       |//! Holds all global state outside `CONFIG` and tokio.
    4|       |
    5|       |use std::collections::hash_map::Iter;
    6|       |use std::collections::HashMap;
    7|       |use std::io::Write;
    8|       |use std::ops::Deref;
    9|       |use std::sync::atomic::{AtomicBool, AtomicI64, AtomicUsize, Ordering};
   10|       |use std::sync::{Arc, RwLock};
   11|       |
   12|       |use chrono::Utc;
   13|       |use futures::task::Task;
   14|       |
   15|       |use log::Level::Debug;
   16|       |use log::*;
   17|       |
   18|       |use crate::config;
   19|       |use crate::config::config::ServerConfig;
   20|       |use crate::errors::ServerError;
   21|       |use crate::init::CONFIG;
   22|       |use crate::init::NGIN;
   23|       |use crate::message::response::{get_response_cc_msg, get_response_stats_msg};
   24|       |use crate::message::stomp_message::StompMessage;
   25|       |use crate::workflow::destination::destination::Destination;
   26|       |
   27|       |// const STATS_TOPIC: &str = "/xtomp/stat";
   28|       |
   29|       |pub struct DestinationServer {
   30|       |    // state
   31|       |    message_id: AtomicUsize,
   32|       |    session_id: AtomicUsize,
   33|       |    destination_id: AtomicUsize,
   34|       |    destinations: HashMap<String, Arc<RwLock<Destination>>>,
   35|       |    destinations_by_id: HashMap<usize, Arc<RwLock<Destination>>>,
   36|       |
   37|       |    // life cycle
   38|       |    shutdown: AtomicBool,
   39|       |    task_set: AtomicBool,
   40|       |    task: RwLock<Option<Task>>,
   41|       |
   42|       |    // stats
   43|       |    pub uptime: AtomicI64,
   44|       |    pub connections_count: AtomicUsize,
   45|       |    pub connections_total: AtomicUsize,
   46|       |    pub messages_total: AtomicUsize,
   47|       |}
   48|       |
   49|       |impl DestinationServer {
   50|      3|    pub(crate) fn new() -> DestinationServer {
   51|      3|        DestinationServer {
   52|      3|            message_id: AtomicUsize::new(DestinationServer::init_id()),
   53|      3|            session_id: AtomicUsize::new(1),
   54|      3|            destination_id: AtomicUsize::new(1),
   55|      3|            destinations: HashMap::new(),
   56|      3|            destinations_by_id: HashMap::new(),
   57|      3|            shutdown: AtomicBool::new(false),
   58|      3|            task_set: AtomicBool::new(false),
   59|      3|            task: RwLock::new(None),
   60|      3|            uptime: AtomicI64::new(Utc::now().timestamp_millis()),
   61|      3|            connections_count: AtomicUsize::new(0),
   62|      3|            connections_total: AtomicUsize::new(0),
   63|      3|            messages_total: AtomicUsize::new(0),
   64|      3|        }
   65|      3|    }
   66|       |
   67|       |    #[cfg(target_pointer_width = "64")]
   68|      3|    fn init_id() -> usize {
   69|      3|        Utc::now().timestamp_millis() as usize
   70|      3|    }
   71|       |
   72|       |    #[cfg(target_pointer_width = "32")]
   73|       |    fn init_id() -> usize {
   74|       |        1
   75|       |    }
   76|       |
   77|       |    // STATE
   78|       |
   79|      9|    pub(crate) fn new_message_id(&self) -> usize {
   80|      9|        // this wraps on overflow
   81|      9|        self.messages_total.fetch_add(1, Ordering::SeqCst);
   82|      9|        return self.message_id.fetch_add(1, Ordering::SeqCst);
   83|      9|    }
   84|       |
   85|      8|    pub(crate) fn new_session(&self) -> usize {
   86|      8|        self.connections_count.fetch_add(1, Ordering::SeqCst);
   87|      8|        self.connections_total.fetch_add(1, Ordering::SeqCst);
   88|      8|        return self.session_id.fetch_add(1, Ordering::SeqCst);
   89|      8|    }
   90|       |
   91|      8|    pub(crate) fn drop_session(&self) -> usize {
   92|      8|        return self.connections_count.fetch_sub(1, Ordering::SeqCst);
   93|      8|    }
   94|       |
   95|      2|    fn create_pid_file(&self) -> std::io::Result<()> {
   96|      2|        let pid = std::process::id().to_string();
   97|      2|        let mut file = std::fs::File::create(&CONFIG.pid_file)?;
   98|      2|        file.write_all(pid.as_bytes())?;
   99|      2|        Ok(())
  100|      2|    }
  101|       |
  102|     12|    fn init_destination(
  103|     12|        &self,
  104|     12|        destination: &config::config::Destination,
  105|     12|    ) -> Arc<RwLock<Destination>> {
  106|     12|        Arc::new(RwLock::new(Destination::new(
  107|     12|            destination.name.clone(),
  108|     12|            self.destination_id.fetch_sub(1, Ordering::SeqCst),
  109|     12|            destination,
  110|     12|        )))
  111|     12|    }
  112|       |
  113|      3|    pub fn find_destination(&self, name: &str) -> Option<&Arc<RwLock<Destination>>> {
  114|      3|        return self.destinations.get(&String::from(name));
  115|      3|    }
  116|       |
  117|       |    /// publish message without security checks, this is an internal API and should not be called from filters.
  118|       |    pub fn publish_message_internal(
  119|       |        &self,
  120|       |        name: &str,
  121|       |        message: StompMessage,
  122|       |    ) -> Result<usize, ServerError> {
  123|      0|        if let Some(destination) = self.destinations.get(&String::from(name)) {
  124|      0|            return destination.write().unwrap().push_owned(message);
  125|      0|        }
  126|      0|        Err(ServerError::DestinationUnknown)
  127|      0|    }
  128|       |
  129|      0|    pub fn find_destination_by_id(&self, id: &usize) -> Option<&Arc<RwLock<Destination>>> {
  130|      0|        return self.destinations_by_id.get(id);
  131|      0|    }
  132|       |
  133|      0|    pub fn iter(&self) -> Iter<String, Arc<RwLock<Destination>>> {
  134|      0|        self.destinations.iter()
  135|      0|    }
  136|       |
  137|       |    // LIFECYCLE
  138|       |
  139|       |    /// called once on boot
  140|      2|    pub(crate) fn init(&mut self, config: &ServerConfig) {
  141|      2|        if let Err(_) = self.create_pid_file() {
  142|      0|            warn!("error creating pid file: {}", std::process::id());
  143|      2|        }
  144|     14|        for (k, v) in &config.destinations {
  145|     12|            let destination = self.init_destination(v);
  146|     12|            let destination_id = destination.read().unwrap().id();
  147|     12|            self.destinations.insert(k.clone(), destination.clone());
  148|     12|            self.destinations_by_id
  149|     12|                .insert(destination_id, destination.clone());
  150|     12|            info!("created destination {}", k);
  151|       |        }
  152|      2|        if config.enable_console {
  153|      1|            NGIN.init();
  154|      1|        }
  155|      2|        debug!("init finished");
  156|      2|    }
  157|       |
  158|       |    /// fired every 60 seconds
  159|      0|    pub(crate) fn tick(&self) -> Result<(), tokio::timer::Error> {
  160|      0|        debug!("server tick");
  161|       |        // bit crap, no real need for these atomic locks except we cannot get mut ref on a static
  162|       |        // TODO we can but its unsafe, need to check this is really single threaded
  163|      0|        if !self.task_set.load(Ordering::Relaxed) {
  164|      0|            self.task_set.store(true, Ordering::Relaxed);
  165|      0|            self.task.write().unwrap().replace(futures::task::current());
  166|      0|            // return OK here so when we boot we don't print stats
  167|      0|            return Ok(());
  168|      0|        }
  169|      0|
  170|      0|        if self.shutdown.load(Ordering::Relaxed) {
  171|      0|            if self.connections_count.load(Ordering::SeqCst) == 0 {
  172|      0|                warn!("shutting down");
  173|      0|                std::process::exit(0);
  174|       |                // TODO tokio Runtime that shutdown gracefully
  175|       |                // return Err(tokio::timer::Error::shutdown());
  176|       |            } else {
  177|      0|                if log_enabled!(Debug) {
  178|      0|                    debug!(
  179|      0|                        "connections still open {}",
  180|      0|                        self.connections_count.load(Ordering::Relaxed)
  181|       |                    );
  182|      0|                }
  183|       |            }
  184|      0|        }
  185|       |
  186|      0|        if let Some(stats_dest) = self.destinations.get(&String::from("/xtomp/stat")) {
  187|       |            {
  188|      0|                if let Ok(mut stats_dest) = stats_dest.write() {
  189|      0|                    debug!("pushing cc message");
  190|      0|                    stats_dest
  191|      0|                        .push(&get_response_cc_msg(
  192|      0|                            self.connections_count.load(Ordering::Relaxed),
  193|      0|                            self.uptime.load(Ordering::Relaxed),
  194|      0|                            self.connections_total.load(Ordering::Relaxed),
  195|      0|                            self.messages_total.load(Ordering::Relaxed),
  196|      0|                        ))
  197|      0|                        .ok(); // swallow errors
  198|      0|                }
  199|       |            }
  200|      0|            for (name, d) in &self.destinations {
  201|      0|                if name.as_str() == "/xtomp/stat" {
  202|      0|                    continue;
  203|      0|                }
  204|      0|                let destination = d.read().unwrap();
  205|      0|                if destination.record_stats() {
  206|      0|                    debug!("pushing stats message for {}", destination.name());
  207|       |                    {
  208|      0|                        if let Ok(mut stats_dest) = stats_dest.write() {
  209|      0|                            stats_dest
  210|      0|                                .push(&get_response_stats_msg(
  211|      0|                                    destination.name(),
  212|      0|                                    destination.max_connections(),
  213|      0|                                    destination.len(),
  214|      0|                                    destination.message_delta(),
  215|      0|                                    destination.message_total(),
  216|      0|                                ))
  217|      0|                                .ok(); // swallow errors
  218|      0|                        }
  219|       |                    }
  220|      0|                }
  221|       |            }
  222|      0|        }
  223|      0|        if CONFIG.enable_console {
  224|      0|            NGIN.set_stats_cc();
  225|      0|            for (i, destination) in self.destinations.values().enumerate() {
  226|      0|                if let Ok(destination) = destination.read() {
  227|      0|                    NGIN.set_stats_destination(
  228|      0|                        i,
  229|      0|                        destination.max_connections(),
  230|      0|                        destination.len(),
  231|      0|                        destination.message_delta(),
  232|      0|                        destination.message_total(),
  233|      0|                    );
  234|      0|                }
  235|       |            }
  236|      0|        }
  237|      0|        info!(
  238|      0|            "stat server sz={}, up={}, Σ={}, Σµ={},",
  239|      0|            self.connections_count.load(Ordering::Relaxed),
  240|      0|            self.uptime.load(Ordering::Relaxed),
  241|      0|            self.connections_total.load(Ordering::Relaxed),
  242|      0|            self.messages_total.load(Ordering::Relaxed)
  243|       |        );
  244|      0|        for (name, d) in &self.destinations {
  245|      0|            if name.as_str() == "/xtomp/stat" {
  246|      0|                continue;
  247|      0|            }
  248|      0|            let destination = d.read().unwrap();
  249|      0|            if destination.record_stats() {
  250|      0|                info!(
  251|      0|                    "stat d={}, s={}, q={}, Δ={}, Σ={}",
  252|      0|                    destination.name(),
  253|      0|                    destination.max_connections(),
  254|      0|                    destination.len(),
  255|      0|                    destination.message_delta(),
  256|      0|                    destination.message_total()
  257|       |                )
  258|      0|            }
  259|       |        }
  260|       |
  261|      0|        Ok(())
  262|      0|    }
  263|       |
  264|      0|    pub fn is_shutdown(&self) -> bool {
  265|      0|        self.shutdown.load(Ordering::Relaxed)
  266|      0|    }
  267|       |
  268|      0|    pub fn shutdown(&self) {
  269|      0|        self.shutdown.store(true, Ordering::Relaxed);
  270|      0|        for (_k, d) in &self.destinations {
  271|      0|            d.write().unwrap().shutdown();
  272|      0|        }
  273|      0|    }
  274|       |
  275|      0|    pub fn notify(&self) {
  276|      0|        let task = self.task.write().unwrap();
  277|      0|        if let Some(task) = task.deref() {
  278|      0|            task.notify();
  279|      0|        }
  280|      0|    }
  281|       |}
  282|       |
  283|       |#[cfg(test)]
  284|       |mod tests {
  285|       |    use crate::config;
  286|       |
  287|       |    use super::*;
  288|       |
  289|      1|    #[test]
  290|      1|    fn test_init() {
  291|      1|        let mut server = DestinationServer::new();
  292|      1|
  293|      1|        let mut config = ServerConfig {
  294|      1|            ..Default::default()
  295|      1|        };
  296|      1|
  297|      1|        let d1 = config::config::Destination {
  298|      1|            name: "memtop-a".to_string(),
  299|      1|            ..Default::default()
  300|      1|        };
  301|      1|        let d2 = config::config::Destination {
  302|      1|            name: "memtop-b".to_string(),
  303|      1|            ..Default::default()
  304|      1|        };
  305|      1|        let destinations = &mut config.destinations;
  306|      1|        destinations.insert(String::from("memtop-a"), d1);
  307|      1|        destinations.insert(String::from("memtop-b"), d2);
  308|      1|
  309|      1|        server.init(&config);
  310|      1|
  311|      1|        match server.find_destination("memtop-a") {
  312|      1|            Some(_d) => {}
  313|      0|            _ => panic!("destination not found"),
  314|       |        }
  315|      1|        match server.find_destination("memtop-b") {
  316|      1|            Some(_d) => {}
  317|      0|            _ => panic!("destination not found"),
  318|       |        }
  319|      1|        match server.find_destination("memtop-c") {
  320|      0|            Some(_d) => panic!("wrong destination found"),
  321|      1|            _ => {}
  322|      1|        }
  323|      1|    }
  324|       |
  325|      1|    #[test]
  326|      1|    fn test_new_id() {
  327|      1|        let server = DestinationServer::new();
  328|      1|        assert_ne!(server.new_message_id(), server.new_message_id());
  329|      1|    }
  330|       |}

src/workflow/http_router.rs:
    1|       |use log::*;
    2|       |
    3|       |use crate::message::stomp_message::StompMessage;
    4|       |use crate::workflow::context::Context;
    5|       |use crate::workflow::filter::http::closure::ClosureFilter;
    6|       |use crate::workflow::filter::http::get::GetFilter;
    7|       |use crate::workflow::filter::http::health_check::HealthCheckFilter;
    8|       |use crate::workflow::filter::http::not_found::NotFoundFilter;
    9|       |use crate::workflow::filter::http::upgrade::UpgradeFilter;
   10|       |use crate::workflow::filter::http::version::VersionFilter;
   11|       |use crate::workflow::filter::{HttpError, HttpFilter};
   12|       |
   13|       |// route an Http message through the Http filters
   14|       |
   15|       |lazy_static! {
   16|       |    pub static ref FILTER_CHAIN_STATIC: FilterChain = FilterChain {
   17|       |        chain: build_filter_chain()
   18|       |    };
   19|       |}
   20|       |
   21|       |// bespoke filter chain additions
   22|       |
   23|       |static mut CHAIN: Vec<(Box<dyn HttpFilter + Sync>, usize)> = Vec::new();
   24|       |static mut INITIALIZED: bool = false;
   25|       |static DEFAULT_INSERT_POINT: usize = 3;
   26|       |
   27|       |/// Insert a bespoke filter into the filter chain, filter supports `init()`
   28|       |///
   29|       |///  # Panics
   30|       |///
   31|       |/// if called after bootstrapping the server.
   32|       |///
   33|      1|pub fn insert_http_filter(filter: Box<dyn HttpFilter + Sync>) {
   34|      1|    unsafe {
   35|      1|        if INITIALIZED {
   36|      0|            panic!("Cannot insert filters after the server is initialized");
   37|      1|        }
   38|      1|        CHAIN.push((filter, DEFAULT_INSERT_POINT));
   39|      1|    }
   40|      1|}
   41|       |
   42|       |/// Voodoo inserts: populate the filter chain at a different point,
   43|       |/// This is not safe across different versions as the default chain changes
   44|       |/// However, with a bit of care it does allow hooking into the filter chain anywhere.
   45|       |///
   46|       |/// # Panics
   47|       |///
   48|       |/// if `at > FILTER_CHAIN_STATIC.len`, which is not known until runtime
   49|      0|pub fn insert_http_filter_at(filter: Box<dyn HttpFilter + Sync>, at: usize) {
   50|      0|    unsafe {
   51|      0|        if INITIALIZED {
   52|      0|            panic!("Cannot insert filters after the server is initialized");
   53|      0|        }
   54|      0|        CHAIN.push((filter, at));
   55|      0|    }
   56|      0|}
   57|       |
   58|       |/// Insert a bespoke filter as a closure into the filter chain
   59|       |///
   60|       |///  # Panics
   61|       |///
   62|       |/// if called after bootstrapping the server.
   63|       |///
   64|      0|pub fn insert_http(
   65|      0|    closure: fn(&mut Context, message: &mut StompMessage) -> Result<bool, HttpError>,
   66|      0|) {
   67|      0|    unsafe {
   68|      0|        if INITIALIZED {
   69|      0|            panic!("Cannot insert filters after the server is initialized");
   70|      0|        }
   71|      0|        insert_http_filter(Box::new(ClosureFilter::new(closure)));
   72|      0|    }
   73|      0|}
   74|       |
   75|       |pub struct FilterChain {
   76|       |    chain: Vec<Box<dyn HttpFilter + Sync>>,
   77|       |}
   78|       |
   79|      0|pub fn build_filter_chain() -> Vec<Box<dyn HttpFilter + Sync>> {
   80|      0|    let mut route: Vec<Box<dyn HttpFilter + Sync>> = vec![
   81|      0|        Box::new(VersionFilter::new()),
   82|      0|        Box::new(UpgradeFilter::new()),
   83|      0|        Box::new(GetFilter::new()),
   84|      0|        // bespoke filters inserted here
   85|      0|        Box::new(HealthCheckFilter::new()),
   86|      0|        Box::new(NotFoundFilter::new()),
   87|      0|    ];
   88|       |
   89|       |    unsafe {
   90|      0|        while CHAIN.len() > 0 {
   91|      0|            let filter = CHAIN.remove(CHAIN.len() - 1);
   92|      0|            route.insert(filter.1, filter.0);
   93|      0|        }
   94|      0|        INITIALIZED = true;
   95|       |    }
   96|       |
   97|      0|    for filter in route.iter_mut() {
   98|      0|        filter.init();
   99|      0|        info!("{:?}.init()", filter);
  100|       |    }
  101|       |
  102|      0|    debug!("HTTP init finished");
  103|      0|    route
  104|      0|}
  105|       |
  106|       |/// route incoming HTTP messages
  107|      0|pub fn route_http(mut ctx: &mut Context, message: &mut StompMessage) -> Result<(), ()> {
  108|      0|    for f in &FILTER_CHAIN_STATIC.chain {
  109|      0|        match f.do_filter(&mut ctx, message) {
  110|      0|            Ok(false) => {
  111|      0|                // loop
  112|      0|            }
  113|       |            Ok(true) => {
  114|      0|                break;
  115|       |            }
  116|      0|            Err(fe) => {
  117|      0|                if let Some(log_message) = fe.log_message {
  118|      0|                    warn!("{}", log_message);
  119|      0|                }
  120|      0|                return Err(());
  121|       |            }
  122|       |        }
  123|       |    }
  124|      0|    return Ok(());
  125|      0|}

src/workflow/sha_auth.rs:
    1|       |use log::*;
    2|       |use sha1::Sha1;
    3|       |
    4|       |/// authentication using hashes based on a shared secret
    5|       |
    6|      2|pub fn sha_auth(
    7|      2|    login: &String,
    8|      2|    token: &String,
    9|      2|    secret: &String,
   10|      2|    secret_timeout: i64,
   11|      2|    now: i64,
   12|      2|) -> Result<String, ()> {
   13|      2|    let mut sha1 = Sha1::new();
   14|      2|    sha1.update(login.as_bytes());
   15|      2|    sha1.update(secret.as_bytes());
   16|      2|    let sha_res = sha1.digest().bytes();
   17|      2|    let encoded = base64::encode(sha_res.as_ref());
   18|      2|    if encoded.eq(token) {
   19|      2|        return check_timeout(login, secret_timeout, now);
   20|       |    } else {
   21|      0|        debug!("sha check failed '{}' '{}'", encoded, token);
   22|       |    }
   23|       |
   24|      0|    Err(())
   25|      2|}
   26|       |
   27|       |/// Check that the login has not timed out, login must be   login + ' ' + timestamp + ' ' + random
   28|      2|fn check_timeout(login: &String, secret_timeout: i64, now: i64) -> Result<String, ()> {
   29|      2|    let mut username = "";
   30|      5|    for (i, part) in login.split_whitespace().enumerate() {
   31|      5|        if i == 0 {
   32|      2|            username = part;
   33|      3|        }
   34|      5|        if i == 1 {
   35|      2|            if let Ok(timestamp) = part.parse::<i64>() {
   36|      2|                if timestamp * 1000 > now - secret_timeout {
   37|      1|                    return Ok(String::from(username));
   38|      1|                }
   39|      0|            }
   40|      3|        }
   41|       |    }
   42|       |
   43|      1|    debug!("timeout check failed");
   44|      1|    Err(())
   45|      2|}
   46|       |
   47|       |#[cfg(test)]
   48|       |mod tests {
   49|       |    use super::*;
   50|       |
   51|      1|    #[test]
   52|      1|    fn test_happy_path() {
   53|      1|        let secret = String::from("XIxoIl6ngolYKQOrXpunRLCMWxR6O0lDI+HycNN4Ffo=");
   54|      1|        // for test purposes set the timestamp of the hash and the current date to zero
   55|      1|        // hash generated with  echo -n "harry 0 randXIxoIl6ngolYKQOrXpunRLCMWxR6O0lDI+HycNN4Ffo=" | sha1sum | awk '{print $1}' | xxd -r -p - - | base64
   56|      1|        assert_eq!(
   57|      1|            "harry",
   58|      1|            sha_auth(
   59|      1|                &String::from("harry 0 rand"),
   60|      1|                &String::from("9hdbYaisq45xkYhCJQCubqzxBZU="),
   61|      1|                &secret,
   62|      1|                60000,
   63|      1|                0
   64|      1|            )
   65|      1|            .unwrap()
   66|      1|        );
   67|      1|    }
   68|       |
   69|      1|    #[test]
   70|      1|    fn test_auth_timeout() {
   71|      1|        let secret = String::from("XIxoIl6ngolYKQOrXpunRLCMWxR6O0lDI+HycNN4Ffo=");
   72|      1|
   73|      1|        assert_eq!(
   74|      1|            Some(()),
   75|      1|            sha_auth(
   76|      1|                &String::from("harry 0 rand"),
   77|      1|                &String::from("9hdbYaisq45xkYhCJQCubqzxBZU="),
   78|      1|                &secret,
   79|      1|                60000,
   80|      1|                60001
   81|      1|            )
   82|      1|            .err()
   83|      1|        );
   84|      1|    }
   85|       |}
```
