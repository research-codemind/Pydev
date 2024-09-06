Licensed under the terms of the Eclipse Public License (EPL).

Please see the license.txt included with this distribution for details.

PyDev is a Python plugin for Eclipse. See http://pydev.org for more details.

To get started in PyDev see: http://pydev.org/manual_101_root.html

For developing PyDev see: http://pydev.org/developers.html

For contact, tracker, sponsorship, see: http://www.pydev.org/about.html

## 빌드 방법
1. feature/codemind_v4.5.8 브랜치 checkout
2. [필수 변경 사항](#필수-변경-사항)들을 적용
3. `mvn intall` 명령 실행
4. 패키지 확인후 내부 Maven 서버로 이관
---

### 2.x 지원 방법
PyDev 10.x.x 부터 Python 2.x 지원을 제거하여 2.x에 대한 데이터 추가가 필요하여 다음과 같은 방법으로 복원 진행

1. 똑같은 리포지토리를 다른 디렉토리에 복제 복제
2. 9.3.0 릴리즈 버전(`e44ace7e590fe9f92fe6ff90fd32af1d33cd6914`)으로 체크아웃
3. `GRAMMAR_PYTHON_VERSION_2` 로 검색하여 검색된 내용을 빌드할 디렉토리에 복사 + 붙여넣기를 후 빌드
---

### 필수 변경 사항
1. plugins/org.python.pydev.parser 프로젝트 org.python.pydev.parser.jython.SimpleNode 클래스에 아래 코드 추가
  ```java
  public int endLine;
  public int endColumn;
  ```
2. plugins/org.python.pydev.parser 프로젝트 org.python.pydev.parser.grammarcommon.JJTPythonGrammarState 클래스의 아래 함수에 코드 추가
  ```java
     private void pushNode(Node n, SimpleNode created, int line, int col) {
        ...
        if (created.beginColumn == 0) {
            created.beginColumn = col;
        }
        // 위 코드 바로 밑에 아래코드 추가
        int endLine = Integer.MAX_VALUE;
        int endColumn = Integer.MAX_VALUE;
        Token lastToken = this.grammar.getJJLastPos();
        if(lastToken!=null) {
            if((created.beginLine == lastToken.endLine && created.beginColumn <= lastToken.endColumn) ||
                    created.beginLine < lastToken.endLine) {
                endLine = lastToken.endLine;
                endColumn = lastToken.endColumn;
            }
        }
        Token curToken = this.grammar.getCurrentToken();
        if(curToken!=null) {
            if((created.beginLine == curToken.endLine && created.beginColumn <= curToken.endColumn) ||
                    created.beginLine < curToken.endLine) {
                if((curToken.endLine == endLine && curToken.endColumn < endColumn) ||
                    curToken.beginLine < endLine) {
                    endLine = curToken.endLine;
                    endColumn = curToken.endColumn;
                }
            }
        }
        created.endLine = endLine;
        created.endColumn = endColumn;
        ...
     }
  ```


3. plugins/org.python.pydev.ast에 PythonNature.java 파일의  **getGrammarVersionFromStr**를 다음과 같이 수정
  ```java
    /**
     * @param grammarVersion a string in the format 2.x or 3.x
     * @return the grammar version as given in IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION
     */
    public static int getGrammarVersionFromStr(String grammarVersion) {
        //Note that we don't have the grammar for all versions, so, we use the one closer to it (which is
        //fine as they're backward compatible).
        switch (grammarVersion) {
          case "2.0":
          case "2.1":
          case "2.2":
          case "2.3":
          case "2.4":
          case "2.5":
              return GRAMMAR_PYTHON_VERSION_2_5;

          case "2.6":
              return GRAMMAR_PYTHON_VERSION_2_6;

          case "2.7":
              return GRAMMAR_PYTHON_VERSION_2_7;
            case "3.0":
            case "3.1":
            case "3.2":
            case "3.3":
            case "3.4":
            case "3.5":
                return GRAMMAR_PYTHON_VERSION_3_5;

            case "3.6":
                return GRAMMAR_PYTHON_VERSION_3_6;
            case "3.7":
                return GRAMMAR_PYTHON_VERSION_3_7;
            case "3.8":
                return GRAMMAR_PYTHON_VERSION_3_8;
            case "3.9":
                return GRAMMAR_PYTHON_VERSION_3_9;
            case "3.10":
                return GRAMMAR_PYTHON_VERSION_3_10;
            case "3.11":
                return GRAMMAR_PYTHON_VERSION_3_11;
            case "3.12":
                return GRAMMAR_PYTHON_VERSION_3_12;

            default:
                break;
        }

        if (grammarVersion != null) {
            if (grammarVersion.startsWith("3")) {
                return LATEST_GRAMMAR_PY3_VERSION;

            } else if (grammarVersion.startsWith("2")) {
                //latest in the 2.x series
                return LATEST_GRAMMAR_PY2_VERSION;
            }
        }

        Log.log("Unable to recognize version: " + grammarVersion + " returning default.");
        return LATEST_GRAMMAR_PY3_VERSION; // Default to python 3 now.
    }
  ```

4. plugins/org.python.pydev.core에 IGrammarVersionProvider.java 파일을 다음과 같은 것들을 추가
  - 상수 필드 추가
    ```java
    /**
     * Constants for knowing the version of a grammar (so that jython 2.1 and python 2.1 can be regarded
     * as GRAMMAR_PYTHON_VERSION_2_1), so, in short, it does not differentiate among the many flavors of python
     *
     * They don't start at 0 because we don't want any accidents... ;-)
     */
    public static final int GRAMMAR_PYTHON_VERSION_2_5 = 11;
    public static final int GRAMMAR_PYTHON_VERSION_2_6 = 12;
    public static final int GRAMMAR_PYTHON_VERSION_2_7 = 13;
    public static final int LATEST_GRAMMAR_PY2_VERSION = GRAMMAR_PYTHON_VERSION_2_7;
    ```
  - GrammarsIterator 수정
    ```java
    /**
     * Just create a new class to initialize those values (we cannot do it in the interface)
     */
    class GrammarsIterator {
    
        public static List<Integer> createList() {
            List<Integer> grammarVersions = new ArrayList<>();
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_5);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_7);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_5);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_6);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_7);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_8);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_9);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_10);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_11);
            grammarVersions.add(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_12);
            return Collections.unmodifiableList(grammarVersions);
        }
    
        public static List<String> createStr() {
            List<String> grammarVersions = new ArrayList<>();
            grammarVersions.add("2.5");
            grammarVersions.add("2.6");
            grammarVersions.add("2.7");
            grammarVersions.add("3.5");
            grammarVersions.add("3.6");
            grammarVersions.add("3.7");
            grammarVersions.add("3.8");
            grammarVersions.add("3.9");
            grammarVersions.add("3.10");
            grammarVersions.add("3.11");
            grammarVersions.add("3.12");
            return Collections.unmodifiableList(grammarVersions);
        }
    
        public static Map<Integer, String> createDict() {
            HashMap<Integer, String> ret = new HashMap<>();
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_5, "2.5");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6, "2.6");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_7, "2.7");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_5, "3.5");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_6, "3.6");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_7, "3.7");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_8, "3.8");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_9, "3.9");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_10, "3.10");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_11, "3.11");
            ret.put(IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_12, "3.12");
            return Collections.unmodifiableMap(ret);
        }
    
        public static Map<String, Integer> createStrToInt() {
            HashMap<String, Integer> ret = new HashMap<>();
            ret.put("2.5", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_5);
            ret.put("2.6", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6);
            ret.put("2.7", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_7);
            ret.put("3.5", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_5);
            ret.put("3.6", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_6);
            ret.put("3.7", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_7);
            ret.put("3.8", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_8);
            ret.put("3.9", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_9);
            ret.put("3.10", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_10);
            ret.put("3.11", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_11);
            ret.put("3.12", IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_12);
            return Collections.unmodifiableMap(ret);
        }
    }
    ```
5. plugins/org.python.pydev.parser에 PyParser.java 파일에서 다음을 수정
  - **getGrammarVersionStr** 메서드 수정
    ```java
    public static String getGrammarVersionStr(int grammarVersion) {
        if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_5) {
            return "grammar: Python 2.5";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6) {
            return "grammar: Python 2.6";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_7) {
            return "grammar: Python 2.7";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_5) {
            return "grammar: Python 3.5";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_6) {
            return "grammar: Python 3.6";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_7) {
            return "grammar: Python 3.7";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_8) {
            return "grammar: Python 3.8";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_9) {
            return "grammar: Python 3.9";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_10) {
            return "grammar: Python 3.10";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_11) {
            return "grammar: Python 3.11";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_3_12) {
            return "grammar: Python 3.12";

        } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_CYTHON) {
            return "grammar: Cython";

        } else {
            return "grammar: unrecognized: " + grammarVersion;
        }
    }
    ```

  - **createGrammar** 메서드 수정
    ```java
    public static IGrammar createGrammar(boolean generateTree, int grammarVersion, char[] charArray) {
        IGrammar grammar;
        FastCharStream in = new FastCharStream(charArray);
        switch (grammarVersion) {
            case IPythonNature.GRAMMAR_PYTHON_VERSION_2_5:
                grammar = new PythonGrammar25(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_2_6:
                grammar = new PythonGrammar26(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_2_7:
                grammar = new PythonGrammar27(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_5:
                grammar = new PythonGrammar30(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_6:
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_7:
                grammar = new PythonGrammar36(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_8:
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_9:
                grammar = new PythonGrammar38(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_10:
                grammar = new PythonGrammar310(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_11:
                grammar = new PythonGrammar311(generateTree, in);
                break;
            case IPythonNature.GRAMMAR_PYTHON_VERSION_3_12:
                grammar = new PythonGrammar312(generateTree, in);
                break;
            //case CYTHON: not treated here (only in reparseDocument).
            default:
                throw new RuntimeException("The grammar specified for parsing is not valid: " + grammarVersion);
        }

        if (ENABLE_TRACING) {
            //grammar has to be generated with debugging info for this to make a difference
            grammar.enable_tracing();
        }
        return grammar;
    }
    ```
6. plugins/org.python.pydev.parser에 PrettyPrinterVisitorV2.java 파일에서 다음을 수정
  - **visitTryExcept** 메서드 수정
    ```java
    @Override
    public Object visitTryExcept(TryExcept node) throws Exception {
        visitTryPart(node, node.body);
        for (excepthandlerType h : node.handlers) {

            startStatementPart();
            beforeNode(h);
            doc.addRequire("except", lastNode);
            this.pushTupleNeedsParens();
            if (h.type != null || h.name != null) {
                doc.addRequire(" ", lastNode);
            }
            if (h.type != null) {
                h.type.accept(this);
            }
            if (h.name != null) {

                if (h.type != null) {
                    int grammarVersion = this.prefs.getGrammarVersion();
                    if (grammarVersion < IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6) {
                        doc.addRequire(",", lastNode);

                    } else if (grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_6
                            || grammarVersion == IGrammarVersionProvider.GRAMMAR_PYTHON_VERSION_2_7) {
                        doc.addRequireOneOf(lastNode, "as", ",");

                    } else { // Python 3.0 or greater
                        doc.addRequire("as", lastNode);
                    }
                }
                h.name.accept(this);
            }
            afterNode(h);
            popTupleNeedsParens();
            this.doc.addRequire(":", lastNode);
            this.doc.addRequireIndent(":", lastNode);
            endStatementPart(lastNode);

            for (stmtType st : h.body) {
                st.accept(this);
            }
            dedent();
        }
        visitOrElsePart(node.orelse, "else");
        return null;
    }
    ```
