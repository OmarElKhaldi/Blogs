---
ID: 112
post_title: >
  Tester une application JavaEE avec
  Arquillian
author: Gérald Quintana
post_date: 2012-03-02 09:28:00
post_excerpt: "<p>Comment tester du code destiné à être exécuté dans un conteneur (Servlet, JPA, Spring, EJB, etc.), c'est à dire tirant partie des services offert par ce conteneur (transactions, base de données,etc.) ? A cette question, Spring répond avec un module dédié aux tests. Cette lacune de J2EE a été résolue avec l'introduction dans Java EE 6 de la notion de « conteneur embarqué », sous entendu embarqué dans un test.</p>"
layout: post
permalink: http://blog.zenika-offres.com/?p=112
published: true
---
<p>Comment tester du code destiné à être exécuté dans un conteneur (Servlet, JPA, Spring, EJB, etc.), c'est à dire tirant partie des services offert par ce conteneur (transactions, base de données,etc.) ? A cette question, Spring répond avec un module dédié aux tests. Cette lacune de J2EE a été résolue avec l'introduction dans Java EE 6 de la notion de « conteneur embarqué », sous entendu embarqué dans un test.</p>
<!--more-->
<p><img src="/wp-content/uploads/2015/07/Arquillian-1.png" alt="Ecosystème Arquillian" title="Ecosystème Arquillian" /></p> <p>L'objectif d'Arquillian est d'aller plus loin dans la simplicité et la concision pour accélérer encore l'écriture de tests JavaEE. Bien que ce soit une projet Red Hat, ce n'est pas juste une extension de JBoss pour les tests unitaires. Arquillian se veut plus généraliste, et supporte d'autres conteneurs comme GlassFish ou Tomcat. C'est ce que nous verrons dans cet article qui montre comment tester avec JUnit un EJB qui utilise JPA avec GlassFish et EclipseLink.</p> <h2>Téléchargement</h2> <p>Arquillian n'est pas, pour l'heure, disponible dans une version finale empaquetée comme on pourrait s'y attendre. Tout les binaires nécessaires se trouvent dans le repository Maven de JBoss, on commence donc par ajouter cela dans notre pom.xml :</p> <pre>     &lt;repositories&gt;         &lt;repository&gt;             &lt;id&gt;jboss.org&lt;/id&gt;             &lt;name&gt;JBoss Repository&lt;/name&gt;             &lt;url&gt;https://repository.jboss.org/nexus/content/groups/public/&lt;/url&gt;         &lt;/repository&gt;     &lt;/repositories&gt; </pre> <p>Puis on va ajouter des dépendances sur les modules GlassFish et JUnit d'Arquillian. La version (1.0.0.Final-SNAPSHOT) peut faire sourire (Final et SNAPSHOT sont antinomiques) ou inquiéter. L'API a été figé il y a plusieurs mois déjà, on peut donc espérer qu'il n'y aura pas de changements  majeurs d'ici la version finale :</p> <pre>     &lt;dependencies&gt;         &lt;dependency&gt;             &lt;groupId&gt;org.jboss.arquillian.container&lt;/groupId&gt;             &lt;artifactId&gt;arquillian-glassfish-embedded-3.1&lt;/artifactId&gt;             &lt;version&gt;${arquillian.version}&lt;/version&gt;             &lt;scope&gt;test&lt;/scope&gt;         &lt;/dependency&gt;         &lt;dependency&gt;             &lt;groupId&gt;org.jboss.arquillian.junit&lt;/groupId&gt;             &lt;artifactId&gt;arquillian-junit-container&lt;/artifactId&gt;             &lt;version&gt;${arquillian.version}&lt;/version&gt;             &lt;scope&gt;test&lt;/scope&gt;         &lt;/dependency&gt; </pre> <p>Évidemment, il nous faudra aussi GlassFish Embedded, un driver JDBC pour accéder à la base de données et JUnit, rien d'extraordinaire en somme :</p> <pre>         &lt;dependency&gt;             &lt;groupId&gt;org.glassfish.extras&lt;/groupId&gt;             &lt;artifactId&gt;glassfish-embedded-all&lt;/artifactId&gt;             &lt;version&gt;3.1&lt;/version&gt;             &lt;scope&gt;test&lt;/scope&gt;         &lt;/dependency&gt;         &lt;dependency&gt;             &lt;groupId&gt;org.apache.derby&lt;/groupId&gt;             &lt;artifactId&gt;derby&lt;/artifactId&gt;             &lt;version&gt;10.8.2.2&lt;/version&gt;             &lt;scope&gt;test&lt;/scope&gt;         &lt;/dependency&gt;         &lt;dependency&gt;             &lt;groupId&gt;junit&lt;/groupId&gt;             &lt;artifactId&gt;junit&lt;/artifactId&gt;             &lt;version&gt;4.10&lt;/version&gt;             &lt;scope&gt;test&lt;/scope&gt;         &lt;/dependency&gt;     &lt;/dependencies&gt; </pre> <p>Il se trouve que les dépendances Arquillian tirent des dépendances dont les numéros de version ne sont pas cohérents entre eux. Pour palier à cela, et maîtriser les numéros de version des sous modules Arquillian, il faudra ajouter des dépendances en scope import :</p> <pre>     &lt;dependencyManagement&gt;         &lt;dependencies&gt; 			&lt;dependency&gt; 				&lt;groupId&gt;org.jboss.arquillian&lt;/groupId&gt; 				&lt;artifactId&gt;arquillian-bom&lt;/artifactId&gt; 				&lt;version&gt;${arquillian.version}&lt;/version&gt; 				&lt;type&gt;pom&lt;/type&gt; 				&lt;scope&gt;import&lt;/scope&gt; 			&lt;/dependency&gt; 			&lt;dependency&gt; 				&lt;groupId&gt;org.jboss.shrinkwrap&lt;/groupId&gt; 				&lt;artifactId&gt;shrinkwrap-bom&lt;/artifactId&gt; 				&lt;version&gt;1.0.0-cr-3&lt;/version&gt; 				&lt;type&gt;pom&lt;/type&gt; 				&lt;scope&gt;import&lt;/scope&gt; 			&lt;/dependency&gt;         &lt;/dependencies&gt;     &lt;/dependencyManagement&gt; </pre> <p>ShrinkWrap est un autre projet de RedHat qui permet de fabriquer des Jars ou des Wars en quelques lignes. Il est très lié à Arquillian, nous verrons son utilité ultérieurement.</p> <p>Au final, on se retrouve avec près d'un trentaine de Jars sur le classpath : C'est beaucoup et ridicule à la fois, ca la plupart des Jars ne contiennent qu'une poignée de classes. Bref, le packaging est pour moi le point noir d'Arquillian : C'est compliqué et peu rassurant. Espérons que la version finale corrigera ce défaut.</p> <h2>Cycle de vie du conteneur</h2> <p>Arquillian lie l'exécution des tests unitaires et le cycle de vie du conteneur : démarrage, deploiements, arrêt... <img src="/wp-content/uploads/2015/07/Arquillian-2.png" alt="Cycle de vie Arquillian" title="Cycle de vie Arquillian" /></p> <p>Pour cela, les classes de test seront annotées @RunWith(Arquillian.class) pour qu'Arquillian puisse s'intégrer dans le flux d'exécution JUnit et orchestre l'interaction avec le conteneur :</p> <pre> @RunWith(Arquillian.class) public class UserDAOTest { </pre> <p>Au tout début des tests, Arquillian va donc démarrer le conteneur (GlassFish Embedded dans notre cas). Dans un fichier arquillian.xml, on spécifie quel conteneur on utiliser et son paramétrage  :</p> <pre> &lt;arquillian&gt;     &lt;engine&gt;         &lt;property name=&quot;deploymentExportPath&quot;&gt;target/arquillian&lt;/property&gt;     &lt;/engine&gt;     &lt;container default=&quot;true&quot; qualifier=&quot;glassfish&quot;&gt;         &lt;configuration&gt;             &lt;property name=&quot;sunResourcesXml&quot;&gt;src/test/resources/glassfish-resources-test.xml&lt;/property&gt;         &lt;/configuration&gt;     &lt;/container&gt; &lt;/arquillian&gt; </pre> <p>Le fichier glassfish-resources-test.xml référencé ci-dessus est une configuration spécifique à Glassfish, on y trouve notamment les services fournis par le conteneur à l'application, comme le pool de connexions à la base de données :</p> <pre> &lt;resources&gt;     &lt;jdbc-connection-pool name=&quot;jdbc/JeeDemoTestPool&quot; res-type=&quot;javax.sql.DataSource&quot;                       datasource-classname=&quot;org.apache.derby.jdbc.EmbeddedXADataSource40&quot;                       pool-resize-quantity=&quot;1&quot; max-pool-size=&quot;5&quot; steady-pool-size=&quot;0&quot;                       statement-timeout-in-seconds=&quot;60&quot; &gt;         &lt;property name=&quot;databaseName&quot; value=&quot;JeeDemoTestDB&quot; /&gt;         &lt;property name=&quot;user&quot; value=&quot;JeeDemoTest&quot; /&gt;         &lt;property name=&quot;password&quot; value=&quot;JeeDemoTest&quot; /&gt;         &lt;property name=&quot;connectionAttributes&quot; value=&quot;create=true&quot; /&gt;     &lt;/jdbc-connection-pool&gt;     &lt;jdbc-resource jndi-name=&quot;jdbc/JeeDemoTestDS&quot; pool-name=&quot;jdbc/JeeDemoTestPool&quot; /&gt; &lt;/resources&gt; </pre> <h2>Déploiement d'une application</h2> <p>Dans « test unitaire », il y a « unitaire » : Pourquoi déployer l'application dans sont intégralité si l'on souhaite ne tester qu'un seul composant ?</p> <p>Pour chaque classe de test, Arquillian permet de déployer une archive (Jar/War/Ear) dont le contenu sera spécifique au test. On peut ainsi ne déployer que le strict minimum (une Servlet), ou bien avoir une configuration spécifique pour un test. Sans oublier que déployer peu, c'est déployer vite !</p> <pre>     @Deployment     public static JavaArchive createTestArchive() {         return ShrinkWrap.create(JavaArchive.class, &quot;UserDAOTest.jar&quot;)                 .addPackage(&quot;com.zenika.jeedemo.model&quot;)                 .addClass(UserDAO.class)                 .addAsManifestResource(&quot;persistence-test.xml&quot;,&quot;persistence.xml&quot;);     } </pre> <p>C'est donc ici qu'intervient ShrinkWrap :  Cet outil permet, via une API « fluent » très efficace, de fabriquer une archive (ici un Jar), d'y mettre de
s classes (addPackage, addClass) ou des fichiers de ressources. La méthode qui produit cette archive est static est annotée @Deployment sera invoquée par Arquillian pour le déploiement, à la façon des méthodes @BeforeClass de JUnit.</p> <p>Évidemment, Arquillian permet aussi de déployer une application dans son intégralité à des fins de tests d'intégration ou d'acceptation. Test d'un EJB et de JPA : Pour chaque méthode de test, Arquillian est capable d'injecter les services et composants du conteneur : on récupère ici l'EJB que l'on souhaite tester, on aurait tout aussi bien pu récupérer une DataSource JDBC ou bien l'EntityManager JPA. :</p> <pre>     @EJB     private UserDAO userDAO;     @Test     public void testFindAll() {         List&lt;User&gt; users=userDAO.findAllUsers();         assertFalse(users.isEmpty());     } </pre> <p>Sans Arquillian, notre classe de test n'étant pas gérée par le conteneur, l'injection de dépendances dans la classe de test ne fonctionne pas. On se retrouve alors contraint de passer par JNDI pour récupérer notre objet.</p> <p>Le UserDAO bénéficie de tous les services proposés par le conteneur (transactions, sécurité, base de données). « Conteneur embarqué » ne signifie pas « Conteneur au rabais », les conditions du test sont on ne peut plus proche de la réalité. La détection des problèmes et leur débogage s'en trouve donc facilitée.</p> <p>Sur un banal PC portable, le démarrage du conteneur GlassFish prend environ 2s, le déploiement et l'initialisation du fragment d'application prend entre 5 et 10s.</p> <h2>Conclusion</h2> <p>Passé le goût un peu amer de la configuration Maven, Arquillian est un outil vraiment efficace, les tests unitaires n'ont plus rien à envier à Spring Test. Plus qu'une simple librairie, c'est un véritable framework. A l'origine Seam fut conçu pour améliorer l'intégration EJB/JSF, Arquillian applique la même recette dans un domaine technique différent : rapprocher les tests automatisées et les conteneurs Java EE.</p>