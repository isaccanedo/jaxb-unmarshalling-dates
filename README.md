## Desmarcando datas usando JAXB

# 1. Introdução
Neste tutorial, veremos como desempacotar objetos de data com formatos diferentes usando JAXB.

Primeiro, vamos cobrir o formato de data do esquema padrão. Em seguida, exploraremos como usar formatos diferentes. Também veremos como podemos lidar com um desafio comum que surge com essas técnicas.

# 2. Esquema para Java Binding
Primeiro, precisamos entender a relação entre o esquema XML e os tipos de dados Java. Em particular, estamos interessados ​​no mapeamento entre um esquema XML e objetos de data Java.

De acordo com o mapeamento de Esquema para Java, há três tipos de dados de Esquema que precisamos levar em consideração: xsd: date, xsd: time e xsd: dateTime. Como podemos ver, todos eles são mapeados para javax.xml.datatype.XMLGregorianCalendar.

Também precisamos entender os formatos padrão para esses tipos de esquema XML. Os tipos de dados xsd: date e xsd: time têm os formatos “AAAA-MM-DD” e “hh: mm: ss”. O formato xsd: dateTime é “AAAA-MM-DDThh: mm: ss” onde “T” é um separador que indica o início da seção de tempo.

# 3. Usando o formato de data do esquema padrão
Vamos construir um exemplo que desempacota objetos de data. Vamos nos concentrar no tipo de dados xsd: dateTime porque é um superconjunto dos outros tipos.

Vamos usar um arquivo XML simples que descreve um livro:

```
<book>
    <title>Book1</title>
    <published>1979-10-21T03:31:12</published>
</book>
```

Queremos mapear o arquivo para o objeto Java Book correspondente:

```
@XmlRootElement(name = "book")
public class Book {

    @XmlElement(name = "title", required = true)
    private String title;

    @XmlElement(name = "published", required = true)
    private XMLGregorianCalendar published;

    @Override
    public String toString() {
        return "[title: " + title + "; published: " + published.toString() + "]";
    }

}
```

Finalmente, precisamos criar um aplicativo cliente que converta os dados XML em objetos Java derivados de JAXB:

```
public static Book unmarshalDates(InputStream inputFile) 
  throws JAXBException {
    JAXBContext jaxbContext = JAXBContext.newInstance(Book.class);
    Unmarshaller jaxbUnmarshaller = jaxbContext.createUnmarshaller();
    return (Book) jaxbUnmarshaller.unmarshal(inputFile);
}
```

No código acima, definimos um JAXBContext que é o ponto de entrada para a API JAXB. Em seguida, usamos um JAXB Unmarshaller em um fluxo de entrada para ler nosso objeto:

Se executarmos o código acima e imprimirmos o resultado, obteremos o seguinte objeto Book:

```
[title: Book1; published: 1979-11-28T02:31:32]
```

Devemos observar que, embora o mapeamento padrão para xsd: dateTime seja XMLGregorianCalendar, também poderíamos ter usado os tipos Java mais comuns: java.util.Date e java.util.Calendar, de acordo com o guia do usuário JAXB.

# 4. Usando um formato de data personalizado
O exemplo acima funciona porque estamos usando o formato de data do esquema padrão, “AAAA-MM-DDThh: mm: ss”.

Mas e se quisermos usar outro formato como “AAAA-MM-DD hh: mm: ss”, eliminando o delimitador “T”? Se substituirmos o delimitador por um caractere de espaço em nosso arquivo XML, o desempacotamento padrão falhará.

### 4.1. Construindo um Adaptador Xml Personalizado
Para usar um formato de data diferente, precisamos definir um XmlAdapter.

Vejamos também como mapear o tipo xsd: dateTime para um objeto java.util.Date com nosso XmlAdapter personalizado:

```
public class DateAdapter extends XmlAdapter<String, Date> {

    private static final String CUSTOM_FORMAT_STRING = "yyyy-MM-dd HH:mm:ss";

    @Override
    public String marshal(Date v) {
        return new SimpleDateFormat(CUSTOM_FORMAT_STRING).format(v);
    }

    @Override
    public Date unmarshal(String v) throws ParseException {
        return new SimpleDateFormat(CUSTOM_FORMAT_STRING).parse(v);
    }

}
```

Neste adaptador, usamos SimpleDateFormat para formatar nossa data. Precisamos ter cuidado, pois SimpleDateFormat não é seguro para threads. Para evitar que vários threads tenham problemas com um objeto SimpleDateFormat compartilhado, estamos criando um novo cada vez que precisamos dele.

### 4.2. As partes internas do XmlAdapter
Como podemos ver, o XmlAdapter possui dois parâmetros de tipo, neste caso, String e Date. O primeiro é o tipo usado dentro do XML e é chamado de tipo de valor. Nesse caso, JAXB sabe como converter um valor XML em uma String. O segundo é chamado de tipo de limite e está relacionado ao valor em nosso objeto Java.

O objetivo de um adaptador é converter entre o tipo de valor e um tipo de limite, de uma forma que o JAXB não pode fazer por padrão.

Para construir um XmlAdapter personalizado, temos que substituir dois métodos: XmlAdapter.marshal() e XmlAdapter.unmarshal().

Durante o desempacotamento, a estrutura de ligação JAXB primeiro desempacota a representação XML em uma String e, em seguida, chama DateAdapter.unmarshal() para adaptar o tipo de valor a uma Date. Durante o empacotamento, a estrutura de ligação JAXB invoca DateAdapter.marshal() para adaptar um Date to String, que é então empacotado para uma representação XML.

4.3. Integrando por meio das anotações JAXB
O DateAdapter funciona como um plugin para JAXB e vamos anexá-lo ao nosso campo de data usando a anotação @XmlJavaTypeAdapter. A anotação @XmlJavaTypeAdapter especifica o uso de um XmlAdapter para desempacotamento personalizado:

```
@XmlRootElement(name = "book")
public class BookDateAdapter {

    // same as before

    @XmlElement(name = "published", required = true)
    @XmlJavaTypeAdapter(DateAdapter.class)
    private Date published;

    // same as before

}
```

Também estamos usando as anotações JAXB padrão: anotações @XmlRootElement e @XmlElement.

Finalmente, vamos executar o novo código:

```
[title: Book1; published: Wed Nov 28 02:31:32 EET 1979]
```

# 5. Datas de desempacotamento em Java 8
Java 8 introduziu uma nova API de data / hora. Aqui, vamos nos concentrar na classe LocalDateTime, que é uma das mais comumente usadas.

### 5.1. Construindo um XmlAdapter baseado em LocalDateTime
Por padrão, JAXB não pode ligar automaticamente um valor xsd: dateTime a um objeto LocalDateTime, independentemente do formato de data. Para converter um valor de data do Esquema XML de ou para um objeto LocalDateTime, precisamos definir outro XmlAdapter semelhante ao anterior:

```
public class LocalDateTimeAdapter extends XmlAdapter<String, LocalDateTime> {

    private DateTimeFormatter dateFormat = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @Override
    public String marshal(LocalDateTime dateTime) {
        return dateTime.format(dateFormat);
    }

    @Override
    public LocalDateTime unmarshal(String dateTime) {
        return LocalDateTime.parse(dateTime, dateFormat);
    }

}
```

Nesse caso, usamos DateTimeFormatter em vez de SimpleDateFormat. O primeiro foi introduzido no Java 8 e é compatível com a nova API Date / Time.

Observe que as operações de conversão podem compartilhar um objeto DateTimeFormatter porque DateTimeFormatter é thread-safe.

### 5.2. Integrando o Novo Adaptador
Agora, vamos substituir o adaptador antigo pelo novo em nossa classe Book e também Date with LocalDateTime:

```
@XmlRootElement(name = "book")
public class BookLocalDateTimeAdapter {

    // same as before

    @XmlElement(name = "published", required = true)
    @XmlJavaTypeAdapter(LocalDateTimeAdapter.class)
    private LocalDateTime published;

    // same as before

}
```

Se executarmos o código acima, obteremos a saída:

```
[title: Book1; published: 1979-11-28T02:31:32]
```

Observe que LocalDateTime.toString() adiciona o delimitador “T” entre a data e a hora.

# 6. Conclusão
Neste tutorial, exploramos datas de desempacotamento usando JAXB.

Primeiro, vimos o esquema XML para mapeamento de tipo de dados Java e criamos um exemplo usando o formato de data do esquema XML padrão.

Em seguida, aprendemos como usar um formato de data personalizado com base em um XmlAdapter personalizado e vimos como lidar com a segurança de thread de SimpleDateFormat.

Por fim, aproveitamos a API Java 8 Date/Time superior e thread-safe e as datas não marcadas com formatos personalizados.